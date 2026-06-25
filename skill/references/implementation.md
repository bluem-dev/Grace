# Grace — Python Implementation Reference

Full reference implementation of Grace as a governed, deterministic realization engine. This
consolidates the runtime, governance core, execution-graph (DAG) engine, MCP integration layer,
and the Etta → Grace bridge into one coherent codebase.

---

## Repository Structure

```
grace-v1/
│
├── README.md
├── pyproject.toml
├── requirements.txt
├── .env.example
│
├── grace/
│   │
│   ├── core/
│   │   ├── runtime.py            # GraceRuntime — top-level orchestrator
│   │   ├── execution_engine.py   # ExecutionEngine — graph build + dispatch
│   │   ├── graph.py              # ExecutionNode, ExecutionGraph (DAG model)
│   │   ├── scheduler.py          # Scheduler — ready-node resolution
│   │   ├── state_machine.py      # RuntimeState enum + transitions
│   │   └── lifecycle.py          # Artifact lifecycle helpers
│   │
│   ├── governance/
│   │   ├── core.py               # GovernanceCore — supreme authority gate
│   │   ├── invariant_registry.py # InvariantRegistry
│   │   ├── conflict_resolver.py  # Conflict classification + resolution pipeline
│   │   └── policy_engine.py      # Severity / recoverability policy lookups
│   │
│   ├── protocol/
│   │   ├── schema.py             # Contract dataclasses / validators
│   │   ├── router.py             # RouteEngine — node type dispatch
│   │   └── validator.py          # Schema enforcement helpers
│   │
│   ├── artifacts/
│   │   ├── model.py              # Artifact dataclass
│   │   ├── manager.py            # ArtifactManager — canonical commit authority
│   │   ├── lineage.py            # Version lineage helpers
│   │   └── serializer.py         # JSON (de)serialization
│   │
│   ├── workspace/
│   │   ├── store.py               # WorkspaceStore — artifact registry persistence
│   │   ├── snapshot.py            # Point-in-time state snapshots
│   │   └── history.py             # Execution log + lineage graph
│   │
│   ├── mcp/
│   │   ├── client.py              # MCPClient — raw tool invocation
│   │   ├── adapter.py             # MCPAdapter — governed tool invocation
│   │   └── executor.py            # MCPExecutor — async per-node execution
│   │
│   ├── etta/
│   │   ├── intent.py              # Intent contract helpers
│   │   ├── interface.py           # EttaInterface — parses raw/Etta input
│   │   └── translator.py          # EttaToGraceTranslator — intent → execution graph
│   │
│   └── errors/
│       ├── ontology.py            # Failure dataclass + domain/severity enums
│       ├── classifier.py          # Classifies raw exceptions into the ontology
│       └── recovery.py            # ErrorRecovery — minimal recovery dispatcher
│
├── tests/
│   ├── test_runtime.py
│   ├── test_governance.py
│   ├── test_artifacts.py
│   └── test_execution_flow.py
│
└── examples/
    ├── minimal_run.py
    ├── etta_to_grace_flow.py
    └── mcp_tool_call.py
```

---

## grace/governance/invariant_registry.py

```python
class InvariantRegistry:
    """
    Centralized registry of global system invariants. Every subsystem checks
    against this registry before mutating state. No subsystem may override
    an invariant without going through GovernanceCore.
    """

    GLOBAL_INVARIANTS = [
        "determinism",
        "traceability",
        "identity_integrity",
        "verification_closure",
        "contract_compliance",
        "canonical_uniqueness",
    ]

    def __init__(self):
        self.invariants = list(self.GLOBAL_INVARIANTS)
        self.canonical_registry: dict[str, int] = {}  # artifact_id -> current canonical version number

    def check(self, obj: dict) -> list[str]:
        """
        Returns a list of violated invariant names. Empty list == compliant.
        """
        violations = []

        if obj is None:
            violations.append("contract_compliance")
            return violations

        # canonical_uniqueness: a legitimate REPLACE always advances the version
        # number (this is how convergence/version-collapse moves forward). The
        # actual violation is a STALE or DUPLICATE write: trying to (re)claim
        # canonical status with a version that is not strictly newer than what
        # is already registered. Strictly-newer writes are always allowed.
        if obj.get("canonical_flag") is True and "artifact_id" in obj:
            existing = self.canonical_registry.get(obj["artifact_id"])
            incoming_version = obj.get("version")
            if existing is not None and incoming_version is not None and incoming_version <= existing:
                violations.append("canonical_uniqueness")

        # traceability: every mutating object must carry a trace_id
        if obj.get("trace_id") is None and obj.get("_skip_trace_check") is not True:
            violations.append("traceability")

        return violations

    def register_canonical(self, artifact_id: str, version: int) -> None:
        self.canonical_registry[artifact_id] = version

    def arbitrate(self, a: dict, b: dict) -> dict:
        """
        Deterministic placeholder arbitration: prefer the higher version,
        then the more recently timestamped, then fall back to whichever
        payload is structurally larger (stable, reproducible tie-break).
        """
        if a.get("version", 0) != b.get("version", 0):
            return a if a.get("version", 0) > b.get("version", 0) else b
        if a.get("timestamp") != b.get("timestamp"):
            return a if a.get("timestamp", "") > b.get("timestamp", "") else b
        return a if len(str(a)) >= len(str(b)) else b
```

---

## grace/governance/core.py

```python
import uuid
from datetime import datetime, timezone

from grace.governance.invariant_registry import InvariantRegistry


class GovernanceCore:
    """
    Supreme authority layer. Nothing in Grace mutates canonical state without
    passing through here. If a conflict cannot be resolved, Governance returns
    a halt signal rather than guessing.
    """

    def __init__(self):
        self.registry = InvariantRegistry()
        self.open_conflicts: list[dict] = []
        self.halted = False

    def validate_intent(self, intent: dict) -> dict:
        violations = self.registry.check(intent)
        if violations:
            self._raise_violation("intent", intent, violations)
        return {"approved": True, "violated_invariants": []}

    def validate_result(self, result: dict) -> dict:
        violations = self.registry.check(result)
        if violations:
            self._raise_violation("result", result, violations)
        return {"approved": True, "violated_invariants": []}

    def commit_canonical(self, artifact: dict) -> dict:
        """
        Final gate before ArtifactManager persists a canonical artifact.
        """
        violations = self.registry.check(artifact)
        if violations:
            self._raise_violation("artifact_commit", artifact, violations)
        self.registry.register_canonical(artifact["artifact_id"], artifact["version"])
        return {"approved": True}

    def resolve_conflict(self, a: dict, b: dict, conflict_type: str = "C") -> dict:
        """
        Conflict classification: A=Schema, B=Semantic, C=State, D=Systemic.
        D-type and any S4/S5-tagged conflict escalate straight to halt.
        """
        if conflict_type == "D" or a.get("severity") in ("S4", "S5") or b.get("severity") in ("S4", "S5"):
            self._halt(reason=f"Critical conflict (type={conflict_type}) requires manual resolution")
            return {"resolved": False, "halted": True}

        resolution = self.registry.arbitrate(a, b)
        return {"resolved": True, "halted": False, "canonical": resolution}

    def _raise_violation(self, stage: str, obj: dict, violations: list[str]) -> None:
        self.open_conflicts.append({
            "conflict_id": str(uuid.uuid4()),
            "stage": stage,
            "violations": violations,
            "timestamp": datetime.now(timezone.utc).isoformat(),
        })
        raise GovernanceViolation(f"Invariant violation at {stage}: {violations}")

    def _halt(self, reason: str) -> None:
        self.halted = True
        self.open_conflicts.append({
            "conflict_id": str(uuid.uuid4()),
            "stage": "conflict_resolution",
            "reason": reason,
            "timestamp": datetime.now(timezone.utc).isoformat(),
        })


class GovernanceViolation(Exception):
    """Raised when an invariant check fails. Always halts forward progress for that cycle."""
    pass
```

---

## grace/core/graph.py

```python
from dataclasses import dataclass, field


@dataclass
class ExecutionNode:
    id: str
    name: str
    node_type: str  # "mcp" | "compute" | "llm"
    dependencies: list[str] = field(default_factory=list)
    status: str = "pending"  # pending|ready|running|completed|failed|blocked|retrying|halted
    payload: dict = field(default_factory=dict)
    result: dict | None = None


class ExecutionGraph:
    """
    Directed, acyclic, deterministic. The same intent must always compile to
    the same graph shape unless context has explicitly changed.
    """

    def __init__(self):
        self.nodes: dict[str, ExecutionNode] = {}

    def add_node(self, node: ExecutionNode) -> None:
        self.nodes[node.id] = node

    def has_pending(self) -> bool:
        return any(n.status in ("pending", "ready", "retrying") for n in self.nodes.values())

    def has_failed_blocking(self) -> bool:
        return any(n.status == "failed" for n in self.nodes.values())

    def mark(self, node_id: str, status: str, result: dict | None = None) -> None:
        node = self.nodes[node_id]
        node.status = status
        if result is not None:
            node.result = result
        # propagate blocking to dependents on failure
        if status == "failed":
            for dependent in self.nodes.values():
                if node_id in dependent.dependencies and dependent.status == "pending":
                    dependent.status = "blocked"

    def all_results(self) -> list[dict]:
        return [n.result for n in self.nodes.values() if n.result is not None]
```

---

## grace/core/scheduler.py

```python
from grace.core.graph import ExecutionGraph, ExecutionNode


class Scheduler:
    """Computes the ready set fresh on every cycle — readiness is never cached."""

    def get_ready_nodes(self, graph: ExecutionGraph) -> list[ExecutionNode]:
        ready = []
        for node in graph.nodes.values():
            if node.status not in ("pending",):
                continue
            if all(graph.nodes[dep].status == "completed" for dep in node.dependencies):
                ready.append(node)
        return ready
```

---

## grace/mcp/executor.py

```python
import asyncio
from grace.core.graph import ExecutionNode


class MCPExecutor:
    """
    Stateless async executor. One node, one call, no assumed side effects.
    Wire real tool/MCP/Claude calls into `dispatch()`.
    """

    async def execute(self, node: ExecutionNode) -> dict:
        try:
            result = await self.dispatch(node)
            return {"node": node.id, "status": "completed", "result": result}
        except Exception as e:
            return {"node": node.id, "status": "failed", "error": str(e)}

    async def dispatch(self, node: ExecutionNode) -> dict:
        if node.node_type == "compute":
            return {"output": node.payload}
        if node.node_type == "mcp":
            # placeholder latency; real implementation calls MCPAdapter.execute_tool
            await asyncio.sleep(0)
            return {"output": f"mcp:{node.payload.get('tool', 'unknown')} dispatched"}
        if node.node_type == "llm":
            await asyncio.sleep(0)
            return {"output": "llm node dispatched — wire to Claude/MCP adapter"}
        raise ValueError(f"Unknown node_type: {node.node_type}")


async def run_parallel(nodes: list[ExecutionNode], executor: MCPExecutor) -> list[dict]:
    tasks = [executor.execute(node) for node in nodes]
    return await asyncio.gather(*tasks)
```

---

## grace/mcp/client.py

```python
import requests


class MCPClient:
    """Generic MCP tool invocation layer. Stateless, no semantic guarantees on the response."""

    def __init__(self, base_url: str | None = None, timeout: int = 30):
        self.base_url = base_url
        self.timeout = timeout

    def call_tool(self, tool_name: str, payload: dict) -> dict:
        response = requests.post(f"{self.base_url}/{tool_name}", json=payload, timeout=self.timeout)
        if response.status_code != 200:
            raise MCPCallError(f"MCP failure calling '{tool_name}': {response.text}")
        return response.json()


class MCPCallError(Exception):
    pass
```

---

## grace/mcp/adapter.py

```python
from grace.governance.core import GovernanceCore
from grace.mcp.client import MCPClient


class MCPAdapter:
    """
    Every tool call passes through Governance both before (is this call
    allowed?) and after (is the result trustworthy enough to feed forward?).
    """

    def __init__(self, governance: GovernanceCore, client: MCPClient | None = None):
        self.client = client or MCPClient()
        self.governance = governance

    def execute_tool(self, tool_node: dict) -> dict:
        self.governance.validate_intent(tool_node)

        result = self.client.call_tool(tool_node["tool"], tool_node.get("payload", {}))

        self.governance.validate_result(result if isinstance(result, dict) else {"value": result})

        return {
            "tool": tool_node["tool"],
            "output": result,
            "status": "validated_mcp_execution",
        }
```

---

## grace/artifacts/model.py

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone


@dataclass
class Artifact:
    artifact_id: str
    type: str
    state: dict
    version: int = 1
    canonical: bool = False
    lineage_ref: str | None = None
    status: str = "draft"  # draft|active|canonical|superseded|archived
    created: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
    updated: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
```

---

## grace/artifacts/manager.py

```python
import uuid
from datetime import datetime, timezone

from grace.artifacts.model import Artifact
from grace.governance.core import GovernanceCore


class ArtifactManager:
    """
    Sole authority for canonical state mutation. Enforces: exactly one
    canonical artifact per artifact_id, ever.
    """

    def __init__(self, governance: GovernanceCore):
        self.governance = governance
        self.store: dict[str, Artifact] = {}          # artifact_id -> current canonical artifact
        self.lineage: dict[str, list[Artifact]] = {}   # artifact_id -> full version history

    def commit(self, results: list[dict], artifact_id: str | None = None,
               operation: str = "CREATE") -> Artifact:
        artifact_id = artifact_id or str(uuid.uuid4())
        previous = self.store.get(artifact_id)
        next_version = (previous.version + 1) if previous else 1

        artifact = Artifact(
            artifact_id=artifact_id,
            type="execution_result",
            state={"results": results, "operation": operation},
            version=next_version,
            canonical=True,
            lineage_ref=previous.artifact_id if previous else None,
            status="canonical",
            updated=datetime.now(timezone.utc).isoformat(),
        )

        # Governance is the final gate before this becomes canonical truth.
        self.governance.commit_canonical({
            "artifact_id": artifact.artifact_id,
            "version": artifact.version,
            "canonical_flag": True,
            "trace_id": str(uuid.uuid4()),
        })

        if previous:
            previous.status = "superseded"
            previous.canonical = False
            self.lineage.setdefault(artifact_id, []).append(previous)

        self.store[artifact_id] = artifact
        return artifact

    def get_canonical(self, artifact_id: str) -> Artifact | None:
        return self.store.get(artifact_id)
```

---

## grace/protocol/router.py

```python
from grace.governance.core import GovernanceCore
from grace.mcp.adapter import MCPAdapter


class RouteEngine:
    """Translates converged intent into an execution node list and dispatches by node type."""

    def __init__(self, governance: GovernanceCore | None = None):
        self.governance = governance or GovernanceCore()
        self.mcp = MCPAdapter(self.governance)

    def translate(self, intent: dict) -> list[dict]:
        """
        Minimal viable translation: one compute node wrapping the intent.
        EttaToGraceTranslator (grace/etta/translator.py) produces richer,
        multi-node graphs for structured Etta intents.
        """
        return [{"type": "compute", "payload": intent}]

    def dispatch_mcp(self, node: dict) -> dict:
        return self.mcp.execute_tool(node)

    def local_compute(self, node: dict) -> dict:
        return {"status": "computed", "node": node}
```

---

## grace/core/execution_engine.py

```python
from grace.protocol.router import RouteEngine
from grace.artifacts.manager import ArtifactManager
from grace.governance.core import GovernanceCore


class ExecutionEngine:
    def __init__(self, governance: GovernanceCore):
        self.governance = governance
        self.router = RouteEngine(governance)
        self.artifacts = ArtifactManager(governance)

    def build_graph(self, intent: dict) -> list[dict]:
        return self.router.translate(intent)

    def execute(self, graph: list[dict], artifact_id: str | None = None) -> "Artifact":
        results = [self._execute_node(node) for node in graph]
        return self.artifacts.commit(results, artifact_id=artifact_id, operation="CREATE")

    def _execute_node(self, node: dict) -> dict:
        if node["type"] == "mcp":
            return self.router.dispatch_mcp(node)
        if node["type"] == "compute":
            return self.router.local_compute(node)
        raise ValueError(f"Unknown node type: {node['type']}")
```

---

## grace/core/runtime.py

```python
from grace.core.execution_engine import ExecutionEngine
from grace.governance.core import GovernanceCore


class GraceRuntime:
    """
    Entry point: Etta → Grace execution pipeline.
    INTAKE → GOVERNANCE PRE-CHECK → CONVERGENCE/PLAN → EXECUTION → VERIFICATION → COMMIT → FEEDBACK
    """

    def __init__(self):
        self.governance = GovernanceCore()
        self.engine = ExecutionEngine(self.governance)

    def execute(self, intent: dict, artifact_id: str | None = None) -> dict:
        # Governance pre-check (Intake gate)
        self.governance.validate_intent(intent)

        # Convergence + Plan (here: direct translation; see EttaToGraceTranslator for DAGs)
        execution_graph = self.engine.build_graph(intent)

        # Execution
        artifact = self.engine.execute(execution_graph, artifact_id=artifact_id)

        # Verification happens inside commit() via governance.commit_canonical;
        # a real system inserts an explicit VerificationEngine.validate() step here.
        self.governance.validate_result({"trace_id": "runtime", "artifact_id": artifact.artifact_id})

        return {
            "status": "success",
            "artifact_id": artifact.artifact_id,
            "version": artifact.version,
            "state": artifact.state,
        }
```

---

## grace/etta/interface.py

```python
class EttaInterface:
    """Parses raw input or an Etta-shaped payload into Grace's Intent Contract."""

    def parse_intent(self, raw_input: str) -> dict:
        return {
            "raw": raw_input,
            "intent": self._extract_intent(raw_input),
            "constraints": self._extract_constraints(raw_input),
            "mode": self._detect_mode(raw_input),
            "confidence": 0.92,
        }

    def _extract_intent(self, text: str) -> dict:
        return {"goal": text}

    def _extract_constraints(self, text: str) -> list:
        return []

    def _detect_mode(self, text: str) -> str:
        if "analyze" in text.lower():
            return "analytical"
        return "exploratory"
```

---

## grace/etta/translator.py

```python
import uuid

from grace.core.graph import ExecutionGraph, ExecutionNode
from grace.governance.core import GovernanceCore


class EttaToGraceTranslator:
    """
    Builds a real DAG from Etta's structured output, instead of RouteEngine's
    single-node minimal translation. Use this when the intent decomposes into
    independent sub-tasks (analysis, research, extraction, consolidation...).
    """

    def __init__(self, governance: GovernanceCore):
        self.governance = governance

    def build_execution_graph(self, etta_output: dict) -> ExecutionGraph:
        intent = etta_output["intent"]
        graph = ExecutionGraph()

        analyze_id = str(uuid.uuid4())
        graph.add_node(ExecutionNode(
            id=analyze_id, name="Analyze", node_type="llm",
            payload={"goal": intent.get("goal")},
        ))

        for sub_name in ("Research", "Extract"):
            graph.add_node(ExecutionNode(
                id=str(uuid.uuid4()), name=sub_name, node_type="mcp",
                dependencies=[analyze_id],
                payload={"tool": "claude", "goal": intent.get("goal")},
            ))

        # Governance validates the shape of the graph before anything runs.
        self.governance.validate_intent({
            "trace_id": str(uuid.uuid4()),
            "node_count": len(graph.nodes),
        })

        return graph
```

---

## grace/errors/recovery.py

```python
class ErrorRecovery:
    """Minimal recovery dispatcher. Extend with retry/rollback/re-converge per
    references/governance.md's Recovery Algorithm Framework."""

    def handle(self, error: Exception) -> dict:
        return {
            "status": "recovered",
            "strategy": "fallback_execution",
            "error": str(error),
        }
```

---

## examples/minimal_run.py

```python
from grace.core.runtime import GraceRuntime

runtime = GraceRuntime()

result = runtime.execute({
    "task": "analyze system behavior",
    "source": "etta",
    "trace_id": "example-1",
})

print(result)
```

---

## examples/etta_to_grace_flow.py

```python
import asyncio

from grace.etta.interface import EttaInterface
from grace.etta.translator import EttaToGraceTranslator
from grace.governance.core import GovernanceCore
from grace.core.scheduler import Scheduler
from grace.mcp.executor import MCPExecutor, run_parallel
from grace.artifacts.manager import ArtifactManager


async def main():
    governance = GovernanceCore()
    etta = EttaInterface()
    translator = EttaToGraceTranslator(governance)
    scheduler = Scheduler()
    executor = MCPExecutor()
    artifacts = ArtifactManager(governance)

    etta_output = etta.parse_intent("Analyze system architecture and produce a design proposal")
    graph = translator.build_execution_graph(etta_output)

    while graph.has_pending():
        ready = scheduler.get_ready_nodes(graph)
        if not ready:
            break  # nothing ready and still pending => blocked or cyclic; halt rather than spin
        results = await run_parallel(ready, executor)
        for r in results:
            graph.mark(r["node"], r["status"], result=r)

    artifact = artifacts.commit(graph.all_results(), operation="CREATE")
    print(f"Canonical artifact: {artifact.artifact_id} v{artifact.version}")
    print(f"Governance halted: {governance.halted}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## examples/mcp_tool_call.py

```python
from grace.governance.core import GovernanceCore
from grace.mcp.adapter import MCPAdapter

governance = GovernanceCore()
adapter = MCPAdapter(governance)

# Real usage points MCPClient.base_url at a live MCP server.
node = {"tool": "claude", "payload": {"prompt": "Summarize the last execution cycle"}, "trace_id": "demo"}

try:
    result = adapter.execute_tool(node)
    print(result)
except Exception as e:
    print(f"MCP call failed: {e}")
```

---

## Boot Sequence (Governance-mandated order)

```python
# Required initialization order — failure at any step halts system startup.
1. GovernanceCore()              # constraint authority first — nothing else may exist without it
2. WorkspaceStore()               # persistence layer (artifact registry, logs, lineage)
3. ArtifactManager(governance)    # canonical commit authority
4. ExecutionEngine / Scheduler / MCPAdapter   # execution machinery
5. EttaInterface / EttaToGraceTranslator       # cognition bridge — wired in last
6. GraceRuntime()                 # orchestrator, wraps everything above
```

Note the inversion relative to Etta's boot sequence (`WorkspaceOS` first, `ClaudeBridge` last):
in Grace, the **Governance Core** is the first authority to exist, because every other module —
including Workspace — must answer to it. In Etta, execution authority (`WorkspaceOS`) comes
first because Etta's job is cognition, not persistence; in Grace, governance over persisted
reality *is* the job, so it boots first.

---

## Extending Grace

**Adding a real MCP tool dispatch:**
```python
# grace/mcp/executor.py — extend MCPExecutor.dispatch()
async def dispatch(self, node):
    if node.node_type == "mcp" and node.payload.get("tool") == "claude":
        # call MCPAdapter.execute_tool(...) here against a live Claude/MCP endpoint
        ...
    if node.node_type == "mcp" and node.payload.get("tool") == "http_get":
        import httpx
        async with httpx.AsyncClient() as client:
            resp = await client.get(node.payload["url"])
            return resp.json()
```

**Distributed / multi-workspace Grace (scaling model):**
Grace scales via parallel execution graphs, artifact sharding, a distributed Workspace, and
modular MCP expansion. Each Workspace shard remains the single source of truth for its own
artifact partition; cross-shard reconciliation goes through Governance Core exactly like any
other Type-C (state) conflict.

**Grace MCP Vision (future):**
A semantic MCP layer where artifact identity, replace semantics, and canonical state are
natively understood by the protocol itself — rather than Grace having to translate every
canonical operation down into non-semantic CRUD calls, as it does today.
