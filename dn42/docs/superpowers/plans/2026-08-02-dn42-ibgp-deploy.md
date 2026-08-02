# DN42 Internal iBGP Deploy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 `/workspace/dn42` 中实现内部 `iBGP` 的 deploy 主路径，支持从 `RoutingSession(kind="ibgp")` 构造 spec、生成 BIRD 配置、下发到节点并回写 `RoutingSession.oper_state`。

**Architecture:** 继续沿用现有内部网络的分层方式：领域层负责 `TransportLink` / `RoutingSession` / `InternalFabricRenderContext`，新增 `iBGP` 专属的 `spec -> render -> deploy payload -> deploy service` 链路。`WireGuard` 继续只负责传输承载，`OSPF` 继续只负责 underlay reachability，`iBGP` 只负责内部 BGP 邻居配置和独立状态回写。

**Tech Stack:** Python 3.11、SQLAlchemy、dataclasses、pytest、ruff、Go 1.22、现有 WSS node protocol、`birdc`

---

## 文件结构

### 需要读取的现有文件

- `/workspace/dn42/docs/superpowers/specs/2026-08-02-dn42-ibgp-deploy-design.md`
- `/workspace/dn42/app/db/models.py`
- `/workspace/dn42/app/internal_fabric/context.py`
- `/workspace/dn42/app/internal_fabric/ospf_spec.py`
- `/workspace/dn42/app/internal_fabric/ospf_deploy.py`
- `/workspace/dn42/internal/config/config.go`
- `/workspace/dn42/internal/runner/runner.go`
- `/workspace/dn42/internal/api/server.go`
- `/workspace/dn42/docs/node-api.md`

### 需要新增的文件

- `/workspace/dn42/app/internal_fabric/ibgp_spec.py`
- `/workspace/dn42/app/internal_fabric/ibgp_render.py`
- `/workspace/dn42/app/internal_fabric/ibgp_deploy_payload.py`
- `/workspace/dn42/app/internal_fabric/ibgp_deploy.py`
- `/workspace/dn42/tests/test_internal_ibgp_spec.py`
- `/workspace/dn42/tests/test_internal_ibgp_render.py`
- `/workspace/dn42/tests/test_internal_ibgp_deploy_payload.py`
- `/workspace/dn42/tests/test_internal_ibgp_deploy_service.py`
- `/workspace/dn42/internal/runner/runner_ibgp_test.go`

### 需要修改的现有文件

- `/workspace/dn42/app/internal_fabric/__init__.py`
- `/workspace/dn42/internal/runner/runner.go`
- `/workspace/dn42/internal/api/server.go`
- `/workspace/dn42/internal/api/server_test.go`
- `/workspace/dn42/docs/node-api.md`

### 本阶段约束

- 第一版只支持 `transport_kind = wireguard`
- 不实现 `ZeroTier` 承载上的 `iBGP`
- 不实现 route-reflector / route-server
- 不修改现有 `internal.deploy`、`ospf.internal.*` 的语义
- 所有新增行为先写失败测试再写实现

### Task 1: 实现 WireGuard iBGP spec builder

**Files:**
- Create: `/workspace/dn42/app/internal_fabric/ibgp_spec.py`
- Create: `/workspace/dn42/tests/test_internal_ibgp_spec.py`
- Modify: `/workspace/dn42/app/internal_fabric/__init__.py`

- [ ] **Step 1: Write the failing test**

创建 `/workspace/dn42/tests/test_internal_ibgp_spec.py`：

```python
from app.internal_fabric.context import InternalFabricRenderContext, RenderLink, RenderNode, RenderSession
from app.internal_fabric.ibgp_spec import build_wireguard_ibgp_specs


def test_build_wireguard_ibgp_specs_creates_node_specs() -> None:
    context = InternalFabricRenderContext(
        nodes=[
            RenderNode("node-a", "tokyo", "10.0.0.1", "4242420001", True, False, True, True),
            RenderNode("node-b", "osaka", "10.0.0.2", "4242420001", True, False, True, True),
        ],
        links=[RenderLink("link-1", "node-a", "node-b", "wireguard", "backbone", "enabled", "active")],
        sessions=[RenderSession("rs-1", "link-1", "ibgp", "enabled", "planned")],
    )

    specs = build_wireguard_ibgp_specs(
        context,
        link_addresses={"link-1": {"node-a": "fd42::1", "node-b": "fd42::2"}},
    )

    assert len(specs) == 2
    tokyo = next(item for item in specs if item.node_name == "tokyo")
    assert tokyo.transport_kind == "wireguard"
    assert tokyo.peers[0].peer_node_name == "osaka"
    assert tokyo.peers[0].peer_address == "fd42::2"


def test_build_wireguard_ibgp_specs_skips_nodes_without_ibgp_capability() -> None:
    context = InternalFabricRenderContext(
        nodes=[
            RenderNode("node-a", "tokyo", "10.0.0.1", "4242420001", True, False, True, True),
            RenderNode("node-b", "osaka", "10.0.0.2", "4242420001", True, False, True, False),
        ],
        links=[RenderLink("link-1", "node-a", "node-b", "wireguard", "backbone", "enabled", "active")],
        sessions=[RenderSession("rs-1", "link-1", "ibgp", "enabled", "planned")],
    )

    specs = build_wireguard_ibgp_specs(
        context,
        link_addresses={"link-1": {"node-a": "fd42::1", "node-b": "fd42::2"}},
    )

    assert specs == []
```

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
cd /workspace/dn42 && python -m pytest tests/test_internal_ibgp_spec.py -q
```

Expected:

- FAIL with `ModuleNotFoundError` for `app.internal_fabric.ibgp_spec`

- [ ] **Step 3: Write minimal implementation**

创建 `/workspace/dn42/app/internal_fabric/ibgp_spec.py`：

```python
from dataclasses import dataclass

from app.internal_fabric.context import InternalFabricRenderContext

RENDER_VERSION = "internal-ibgp-v1"


@dataclass(frozen=True)
class InternalIbgpPeerSpec:
    routing_session_id: str
    peer_node_id: str
    peer_node_name: str
    transport_kind: str
    local_asn: str
    peer_asn: str
    local_address: str
    peer_address: str
    topology_role: str


@dataclass(frozen=True)
class InternalIbgpNodeSpec:
    node_id: str
    node_name: str
    router_id: str
    transport_kind: str
    peers: list[InternalIbgpPeerSpec]
    render_version: str


def build_wireguard_ibgp_specs(
    context: InternalFabricRenderContext,
    *,
    link_addresses: dict[str, dict[str, str]],
) -> list[InternalIbgpNodeSpec]:
    nodes_by_id = {node.node_id: node for node in context.nodes}
    links_by_id = {link.link_id: link for link in context.links if link.kind == "wireguard"}
    node_peers: dict[str, list[InternalIbgpPeerSpec]] = {}

    for session in context.sessions:
        if session.kind != "ibgp" or session.admin_state != "enabled":
            continue
        link = links_by_id.get(session.transport_link_id)
        if link is None or link.admin_state != "enabled":
            continue
        source = nodes_by_id[link.source_node_id]
        target = nodes_by_id[link.target_node_id]
        if not source.supports_ibgp or not target.supports_ibgp:
            continue
        addresses = link_addresses.get(link.link_id)
        if not addresses:
            raise ValueError(f"{link.link_id} missing ibgp link addresses")
        source_addr = addresses.get(source.node_id)
        target_addr = addresses.get(target.node_id)
        if not source_addr or not target_addr:
            raise ValueError(f"{link.link_id} missing node address for ibgp")

        node_peers.setdefault(source.node_id, []).append(
            InternalIbgpPeerSpec(
                routing_session_id=session.session_id,
                peer_node_id=target.node_id,
                peer_node_name=target.name,
                transport_kind="wireguard",
                local_asn=source.internal_asn,
                peer_asn=target.internal_asn,
                local_address=source_addr,
                peer_address=target_addr,
                topology_role=link.topology_role,
            )
        )
        node_peers.setdefault(target.node_id, []).append(
            InternalIbgpPeerSpec(
                routing_session_id=session.session_id,
                peer_node_id=source.node_id,
                peer_node_name=source.name,
                transport_kind="wireguard",
                local_asn=target.internal_asn,
                peer_asn=source.internal_asn,
                local_address=target_addr,
                peer_address=source_addr,
                topology_role=link.topology_role,
            )
        )

    specs = []
    for node_id, peers in sorted(node_peers.items(), key=lambda item: nodes_by_id[item[0]].name):
        node = nodes_by_id[node_id]
        specs.append(
            InternalIbgpNodeSpec(
                node_id=node.node_id,
                node_name=node.name,
                router_id=node.internal_router_id,
                transport_kind="wireguard",
                peers=peers,
                render_version=RENDER_VERSION,
            )
        )
    return specs
```

- [ ] **Step 4: Run test to verify it passes**

Run:

```bash
cd /workspace/dn42 && python -m pytest tests/test_internal_ibgp_spec.py -q
```

Expected:

- PASS

- [ ] **Step 5: Commit**

```bash
cd /workspace/dn42 && git add app/internal_fabric/ibgp_spec.py tests/test_internal_ibgp_spec.py app/internal_fabric/__init__.py && git commit -m "feat(internal-fabric): add wireguard ibgp spec builder"
```

### Task 2: 实现 iBGP 渲染与 deploy payload

**Files:**
- Create: `/workspace/dn42/app/internal_fabric/ibgp_render.py`
- Create: `/workspace/dn42/app/internal_fabric/ibgp_deploy_payload.py`
- Create: `/workspace/dn42/tests/test_internal_ibgp_render.py`
- Create: `/workspace/dn42/tests/test_internal_ibgp_deploy_payload.py`
- Modify: `/workspace/dn42/app/internal_fabric/__init__.py`

- [ ] **Step 1: Write the failing test**

创建 `/workspace/dn42/tests/test_internal_ibgp_render.py`：

```python
from app.internal_fabric.ibgp_render import render_ibgp_node_config
from app.internal_fabric.ibgp_spec import InternalIbgpNodeSpec, InternalIbgpPeerSpec


def test_render_ibgp_node_config_renders_bgp_neighbors() -> None:
    node_spec = InternalIbgpNodeSpec(
        node_id="node-a",
        node_name="tokyo",
        router_id="10.0.0.1",
        transport_kind="wireguard",
        peers=[
            InternalIbgpPeerSpec(
                routing_session_id="rs-1",
                peer_node_id="node-b",
                peer_node_name="osaka",
                transport_kind="wireguard",
                local_asn="4242420001",
                peer_asn="4242420001",
                local_address="fd42::1",
                peer_address="fd42::2",
                topology_role="backbone",
            )
        ],
        render_version="internal-ibgp-v1",
    )

    text = render_ibgp_node_config(node_spec)

    assert "protocol bgp IBGPI_wireguard_tokyo_osaka" in text
    assert "local as 4242420001;" in text
    assert "neighbor fd42::2 as 4242420001;" in text
```

创建 `/workspace/dn42/tests/test_internal_ibgp_deploy_payload.py`：

```python
from app.internal_fabric.ibgp_deploy_payload import build_internal_ibgp_deploy_payload
from app.internal_fabric.ibgp_spec import InternalIbgpNodeSpec, InternalIbgpPeerSpec


def test_build_internal_ibgp_deploy_payload_contains_rendered_config() -> None:
    node_spec = InternalIbgpNodeSpec(
        node_id="node-a",
        node_name="tokyo",
        router_id="10.0.0.1",
        transport_kind="wireguard",
        peers=[
            InternalIbgpPeerSpec(
                routing_session_id="rs-1",
                peer_node_id="node-b",
                peer_node_name="osaka",
                transport_kind="wireguard",
                local_asn="4242420001",
                peer_asn="4242420001",
                local_address="fd42::1",
                peer_address="fd42::2",
                topology_role="backbone",
            )
        ],
        render_version="internal-ibgp-v1",
    )

    payload = build_internal_ibgp_deploy_payload(node_spec, request_id="req-1")

    assert payload.config_name == "IBGPI_wireguard_tokyo"
    assert payload.transport_kind == "wireguard"
    assert "protocol bgp IBGPI_wireguard_tokyo_osaka" in payload.bird_config
    assert payload.as_payload()["request_id"] == "req-1"
```

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
cd /workspace/dn42 && python -m pytest tests/test_internal_ibgp_render.py tests/test_internal_ibgp_deploy_payload.py -q
```

Expected:

- FAIL because `ibgp_render.py` and `ibgp_deploy_payload.py` do not exist

- [ ] **Step 3: Write minimal implementation**

创建 `/workspace/dn42/app/internal_fabric/ibgp_render.py`：

```python
from app.internal_fabric.ibgp_spec import InternalIbgpNodeSpec


def config_name(node_spec: InternalIbgpNodeSpec) -> str:
    return f"IBGPI_{node_spec.transport_kind}_{node_spec.node_name}"


def render_ibgp_node_config(node_spec: InternalIbgpNodeSpec) -> str:
    lines = [f"# Generated internal iBGP config for {node_spec.node_name}"]
    for peer in node_spec.peers:
        lines.extend(
            [
                f"protocol bgp {config_name(node_spec)}_{peer.peer_node_name} {{",
                f"  local as {peer.local_asn};",
                f"  neighbor {peer.peer_address} as {peer.peer_asn};",
                f"  source address {peer.local_address};",
                "}",
            ]
        )
    return "\n".join(lines) + "\n"
```

创建 `/workspace/dn42/app/internal_fabric/ibgp_deploy_payload.py`：

```python
from dataclasses import asdict, dataclass

from app.internal_fabric.ibgp_render import config_name, render_ibgp_node_config
from app.internal_fabric.ibgp_spec import InternalIbgpNodeSpec, InternalIbgpPeerSpec


@dataclass(frozen=True)
class InternalIbgpDeployPayload:
    request_id: str
    node: str
    transport_kind: str
    config_name: str
    render_version: str
    bird_config: str
    peers: list[InternalIbgpPeerSpec]

    def as_payload(self) -> dict[str, object]:
        return {
            "request_id": self.request_id,
            "node": self.node,
            "transport_kind": self.transport_kind,
            "config_name": self.config_name,
            "render_version": self.render_version,
            "bird_config": self.bird_config,
            "peers": [asdict(peer) for peer in self.peers],
        }


def build_internal_ibgp_deploy_payload(
    node_spec: InternalIbgpNodeSpec,
    *,
    request_id: str,
) -> InternalIbgpDeployPayload:
    return InternalIbgpDeployPayload(
        request_id=request_id,
        node=node_spec.node_name,
        transport_kind=node_spec.transport_kind,
        config_name=config_name(node_spec),
        render_version=node_spec.render_version,
        bird_config=render_ibgp_node_config(node_spec),
        peers=list(node_spec.peers),
    )
```

- [ ] **Step 4: Run test to verify it passes**

Run:

```bash
cd /workspace/dn42 && python -m pytest tests/test_internal_ibgp_render.py tests/test_internal_ibgp_deploy_payload.py -q
```

Expected:

- PASS

- [ ] **Step 5: Commit**

```bash
cd /workspace/dn42 && git add app/internal_fabric/ibgp_render.py app/internal_fabric/ibgp_deploy_payload.py tests/test_internal_ibgp_render.py tests/test_internal_ibgp_deploy_payload.py app/internal_fabric/__init__.py && git commit -m "feat(internal-fabric): add internal ibgp render and deploy payload"
```

### Task 3: 实现后端 iBGP deploy service 和状态回写

**Files:**
- Create: `/workspace/dn42/app/internal_fabric/ibgp_deploy.py`
- Create: `/workspace/dn42/tests/test_internal_ibgp_deploy_service.py`
- Modify: `/workspace/dn42/app/internal_fabric/__init__.py`

- [ ] **Step 1: Write the failing test**

创建 `/workspace/dn42/tests/test_internal_ibgp_deploy_service.py`：

```python
from app.db.models import Node, NodeCapability, RoutingSession, TransportLink
from app.internal_fabric.ibgp_deploy import deploy_internal_ibgp_node, fetch_internal_ibgp_status, remove_internal_ibgp_node
from app.internal_fabric.ibgp_spec import InternalIbgpNodeSpec, InternalIbgpPeerSpec


def test_deploy_internal_ibgp_node_marks_session_active(db_session, monkeypatch):
    node = Node(name="tokyo", location="Tokyo", url="tokyo.example.net", enabled=True)
    peer = Node(name="osaka", location="Osaka", url="osaka.example.net", enabled=True)
    db_session.add_all([node, peer])
    db_session.commit()
    db_session.add_all([
        NodeCapability(node_id=node.id, supports_internal_wg=True, supports_ospf=True, supports_ibgp=True),
        NodeCapability(node_id=peer.id, supports_internal_wg=True, supports_ospf=True, supports_ibgp=True),
    ])
    link = TransportLink(source_node_id=node.id, target_node_id=peer.id, kind="wireguard")
    db_session.add(link)
    db_session.commit()
    session = RoutingSession(transport_link_id=link.id, kind="ibgp", admin_state="enabled")
    db_session.add(session)
    db_session.commit()

    spec = InternalIbgpNodeSpec(
        node_id=node.id,
        node_name="tokyo",
        router_id="10.0.0.1",
        transport_kind="wireguard",
        peers=[
            InternalIbgpPeerSpec(
                routing_session_id=session.id,
                peer_node_id=peer.id,
                peer_node_name="osaka",
                transport_kind="wireguard",
                local_asn="4242420001",
                peer_asn="4242420001",
                local_address="fd42::1",
                peer_address="fd42::2",
                topology_role="backbone",
            )
        ],
        render_version="internal-ibgp-v1",
    )

    monkeypatch.setattr(
        "app.internal_fabric.ibgp_deploy.request_node_sync",
        lambda node, command, payload, timeout: {"ok": True, "applied": True, "output": "configured"},
    )

    result = deploy_internal_ibgp_node(db_session, node=node, node_spec=spec)

    db_session.refresh(session)
    assert result["ok"] is True
    assert session.oper_state == "active"
```

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
cd /workspace/dn42 && python -m pytest tests/test_internal_ibgp_deploy_service.py -q
```

Expected:

- FAIL because `ibgp_deploy.py` does not exist

- [ ] **Step 3: Write minimal implementation**

创建 `/workspace/dn42/app/internal_fabric/ibgp_deploy.py`：

```python
from __future__ import annotations

import json
import uuid
from typing import Any

from sqlalchemy.orm import Session

from app.db.models import Node, RoutingSession
from app.internal_fabric.ibgp_deploy_payload import build_internal_ibgp_deploy_payload
from app.internal_fabric.ibgp_render import config_name
from app.internal_fabric.ibgp_spec import InternalIbgpNodeSpec
from app.internal_fabric.service import record_snapshot
from app.node_ws import request_node_sync


class InternalIbgpDeployError(RuntimeError):
    pass


def _write_session_state(db: Session, *, node_spec: InternalIbgpNodeSpec, oper_state: str, result: dict[str, Any]) -> None:
    session_ids = {peer.routing_session_id for peer in node_spec.peers}
    for session in db.query(RoutingSession).filter(RoutingSession.id.in_(session_ids)).all():
        session.oper_state = oper_state
    db.commit()
    record_snapshot(
        db,
        node_id=node_spec.node_id,
        snapshot_type="internal_ibgp_deploy",
        status_json=json.dumps(result, sort_keys=True),
    )


def deploy_internal_ibgp_node(db: Session, *, node: Node, node_spec: InternalIbgpNodeSpec, timeout: float = 20.0) -> dict[str, Any]:
    if not node.enabled:
        raise InternalIbgpDeployError("Node is disabled")
    payload = build_internal_ibgp_deploy_payload(node_spec, request_id=uuid.uuid4().hex)
    result = request_node_sync(node, "ibgp.internal.deploy", payload.as_payload(), timeout)
    _write_session_state(db, node_spec=node_spec, oper_state="active" if result.get("ok") else "failed", result=result)
    return result


def remove_internal_ibgp_node(db: Session, *, node: Node, node_spec: InternalIbgpNodeSpec, timeout: float = 20.0) -> dict[str, Any]:
    result = request_node_sync(node, "ibgp.internal.remove", {"request_id": uuid.uuid4().hex, "config_name": config_name(node_spec)}, timeout)
    _write_session_state(db, node_spec=node_spec, oper_state="planned", result=result)
    return result


def fetch_internal_ibgp_status(*, node: Node, node_spec: InternalIbgpNodeSpec, timeout: float = 10.0) -> dict[str, Any]:
    return request_node_sync(node, "ibgp.internal.status", {"config_name": config_name(node_spec)}, timeout)
```

- [ ] **Step 4: Run test to verify it passes**

Run:

```bash
cd /workspace/dn42 && python -m pytest tests/test_internal_ibgp_deploy_service.py -q
```

Expected:

- PASS

- [ ] **Step 5: Commit**

```bash
cd /workspace/dn42 && git add app/internal_fabric/ibgp_deploy.py tests/test_internal_ibgp_deploy_service.py app/internal_fabric/__init__.py && git commit -m "feat(internal-fabric): add internal ibgp deploy service"
```

### Task 4: 扩展节点 runner 和 API 分发

**Files:**
- Modify: `/workspace/dn42/internal/runner/runner.go`
- Create: `/workspace/dn42/internal/runner/runner_ibgp_test.go`
- Modify: `/workspace/dn42/internal/api/server.go`
- Modify: `/workspace/dn42/internal/api/server_test.go`
- Modify: `/workspace/dn42/docs/node-api.md`

- [ ] **Step 1: Write the failing test**

创建 `/workspace/dn42/internal/runner/runner_ibgp_test.go`：

```go
package runner

import (
	"os"
	"path/filepath"
	"testing"
	"time"
)

func TestDeployInternalIbgpRejectsInvalidConfigName(t *testing.T) {
	r := Runner{BirdInternalDir: t.TempDir(), Timeout: time.Second}
	result := r.DeployInternalIbgp(InternalIbgpDeployRequest{
		RequestID:     "req-1",
		Node:          "tokyo",
		TransportKind: "wireguard",
		ConfigName:    "../escape",
		RenderVersion: "internal-ibgp-v1",
		BirdConfig:    "protocol bgp test {}",
	})
	if result.OK || result.Applied {
		t.Fatalf("expected validation failure, got %+v", result)
	}
}

func TestDeployInternalIbgpWritesConfigAndReloadsBird(t *testing.T) {
	dir := t.TempDir()
	birdc := filepath.Join(dir, "birdc")
	if err := os.WriteFile(birdc, []byte("#!/bin/sh\nexit 0\n"), 0o755); err != nil {
		t.Fatalf("write fake birdc: %v", err)
	}

	r := Runner{BirdcPath: birdc, BirdInternalDir: dir, Timeout: time.Second}
	result := r.DeployInternalIbgp(InternalIbgpDeployRequest{
		RequestID:     "req-1",
		Node:          "tokyo",
		TransportKind: "wireguard",
		ConfigName:    "IBGPI_wireguard_tokyo",
		RenderVersion: "internal-ibgp-v1",
		BirdConfig:    "protocol bgp test {}\n",
	})
	if !result.OK || !result.Applied {
		t.Fatalf("expected success, got %+v", result)
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
cd /workspace/dn42 && go test ./internal/runner ./internal/api
```

Expected:

- FAIL because `DeployInternalIbgp` and `ibgp.internal.*` routes do not exist

- [ ] **Step 3: Write minimal implementation**

在 `/workspace/dn42/internal/runner/runner.go` 中新增：

```go
type InternalIbgpDeployRequest struct {
	RequestID     string `json:"request_id"`
	Node          string `json:"node"`
	TransportKind string `json:"transport_kind"`
	ConfigName    string `json:"config_name"`
	RenderVersion string `json:"render_version"`
	BirdConfig    string `json:"bird_config"`
}

func (r Runner) DeployInternalIbgp(req InternalIbgpDeployRequest) DeployResult {
	req.Node = strings.TrimSpace(req.Node)
	req.TransportKind = strings.TrimSpace(req.TransportKind)
	req.ConfigName = strings.TrimSpace(req.ConfigName)
	req.RenderVersion = strings.TrimSpace(req.RenderVersion)
	req.BirdConfig = strings.TrimSpace(req.BirdConfig)
	if err := validateInternalIbgpDeployRequest(req); err != nil {
		return DeployResult{OK: false, Applied: false, Output: err.Error()}
	}
	file := filepath.Join(r.BirdInternalDir, req.ConfigName+".conf")
	if err := ensureChildPath(r.BirdInternalDir, file); err != nil {
		return DeployResult{OK: false, Applied: false, Output: err.Error()}
	}
	if err := writeConfigFile(file, req.BirdConfig+"\n"); err != nil {
		return DeployResult{OK: false, Applied: false, Output: err.Error(), Files: []string{file}}
	}
	reload := r.run(r.BirdcPath, "configure")
	if !reload.OK {
		return DeployResult{OK: false, Applied: true, Output: reload.Output, Files: []string{file}, ConfigName: req.ConfigName}
	}
	return DeployResult{OK: true, Applied: true, Output: reload.Output, Files: []string{file}, ConfigName: req.ConfigName}
}
```

在 `/workspace/dn42/internal/api/server.go` 中新增：

```go
case "ibgp.internal.deploy":
	var req runner.InternalIbgpDeployRequest
	if err := decodeCommandPayload(payload, &req); err != nil {
		return nil, err
	}
	return s.deployInternalIbgp(req), nil
```

- [ ] **Step 4: Run test to verify it passes**

Run:

```bash
cd /workspace/dn42 && gofmt -w internal/runner/runner.go internal/runner/runner_ibgp_test.go internal/api/server.go internal/api/server_test.go && go test ./internal/...
```

Expected:

- PASS

- [ ] **Step 5: Commit**

```bash
cd /workspace/dn42 && git add internal/runner/runner.go internal/runner/runner_ibgp_test.go internal/api/server.go internal/api/server_test.go docs/node-api.md && git commit -m "feat(node): add internal ibgp runner commands"
```

## 自检

### Spec 覆盖

- `InternalIbgpPeerSpec`：Task 1
- `InternalIbgpNodeSpec`：Task 1
- `InternalIbgpDeployPayload`：Task 2
- 后端 `ibgp.internal.deploy`：Task 3
- `RoutingSession.oper_state` 回写：Task 3
- 节点 `ibgp.internal.*`：Task 4
- Node API 文档：Task 4

### 占位词检查

- 没有 `TBD`
- 没有 `TODO`
- 没有空任务

### 命名一致性

- `InternalIbgpPeerSpec`
- `InternalIbgpNodeSpec`
- `build_wireguard_ibgp_specs()`
- `InternalIbgpDeployPayload`
- `build_internal_ibgp_deploy_payload()`
- `deploy_internal_ibgp_node()`
- `remove_internal_ibgp_node()`
- `fetch_internal_ibgp_status()`
- `DeployInternalIbgp()`

以上命名在任务中保持一致。
