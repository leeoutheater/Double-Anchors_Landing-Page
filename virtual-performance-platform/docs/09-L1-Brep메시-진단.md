# 09. L1 standalone — Brep→Mesh 변환 실패 진단 및 패치

문서버전: v1.0 · 작성기준일: 2026-06-01 · 대상: `L1-web-app/standalone/index.html`
(build 2026-05-23.BB, 약 6,529줄) — 드라이브 진행분 직접 분석.

> 결정 반영: **standalone HTML 단일 트랙 계승(F9)**. 본 진단은 그 트랙을 안정화한다.

---

## 1. 증상 (Symptom)

`3D PROSCENIUM_..._Solid Surface_Explode.3dm`(객체 87개: Extrusion 1 + Brep 86)
업로드 시 **86개 Brep 전량 변환 실패**, 뷰포트에 거의 아무것도 표시되지 않음.
진단 로그: `face.getMesh()=null`, `Mesh.createFromBrep 함수 없음`,
`brep.faces.count=0`, 5개 전략(A~E) 모두 실패.

## 2. 근본 원인 (Root Cause) — 코드 버그가 아니라 라이브러리 한계

코드(`brepToMesh`, `safeFaces`, `safeCount`)는 견고하다. 실패는 다음 구조적
원인 때문이다.

1. **rhino3dm.js는 NURBS 메싱을 못 한다 (핵심).**
   rhino3dm은 *지오메트리 I/O* 라이브러리이지 *메싱 커널*이 아니다.
   `Mesh.createFromBrep`는 rhino3dm에 **존재하지 않는다**(API probe 결과 노출
   메서드: `createFromSubDControlNet`, `toThreejsJSONMerged`,
   `createFromThreejsJSON` 뿐). → 전략 B는 원천적으로 불가.
   NURBS surface 평가(pointAt)·제어점 기반 전략 C·D도 trimmed Brep에는 신뢰 불가.
2. **파일에 렌더 메시가 저장돼 있지 않다.** → 전략 A(`face.getMesh`)가 null.
   ("Save Small" 또는 렌더 메시 미저장 상태로 저장된 .3dm)
3. **Explode된 surface들이 이 rhino3dm 빌드에서 `brep.faces().count=0`** 으로
   반환 → face 루프 전략(A·C·D)이 0회 순회.
4. **이미 있는 서버측 경로가 뷰어에 연결돼 있지 않다.**
   `convert-server`(`http://127.0.0.1:8082`) + **Rhino Compute(:8081)** 가 존재하나,
   `/convert/3dm-to-mvr`(MVR 내보내기)에만 쓰이고 **뷰어 표시(brepToMesh)** 에는
   미연결. 뷰어는 client-only로 남아 한계에 부딪힘.

> 한 줄: **"브라우저만으로 NURBS를 메시로 만들 수 없다. 메싱은 Rhino Compute가
> 해야 하는데, 그 경로가 뷰어에 연결돼 있지 않다."**

## 3. 권장 수정 (Primary Fix) — 뷰어를 기존 Rhino Compute에 연결

인프라의 90%가 이미 있다. 뷰어 표시 경로를 convert-server(:8082)/Compute(:8081)에
연결한다.

### 3.1 클라이언트 (index.html) — 서버 메시 폴백 추가

`brepToMesh`/`extrusionToMesh` 실패 시(또는 `conv-mode=compute`일 때) .3dm 원본을
서버로 보내 GLB로 메시화해 받아온다. (`CONVERT_SERVER_URL` 상수 재사용)

```js
// [추가] 파일 파싱 유틸 근처에 삽입
async function meshViaConvertServer(buffer, { mode = 'compute' } = {}) {
  const form = new FormData();
  form.append('file', new Blob([buffer], { type: 'application/octet-stream' }), 'model.3dm');
  form.append('mode', mode);     // 'compute'(정밀) | 'auto' | 'fallback'
  form.append('format', 'glb');
  const r = await fetch(CONVERT_SERVER_URL + '/convert/3dm-to-glb', {
    method: 'POST', body: form, signal: AbortSignal.timeout(120000),
  });
  if (!r.ok) throw new Error(`convert-server ${r.status}`);
  return await r.arrayBuffer(); // GLB 바이트
}
```

로드 플로우(클라이언트 파싱 직후, `convertedAsMesh === 0` 또는 사용자가 compute
모드 선택 시):

```js
if (convertedAsMesh === 0 && _convertServerStatus === 'ok') {
  try {
    const glb = await meshViaConvertServer(buffer, { mode: 'compute' });
    const loader = new THREE.GLTFLoader();           // addons에서 import 필요
    loader.parse(glb, '', (gltf) => scene.add(gltf.scene),
                 (err) => console.warn('GLB 로드 실패', err));
  } catch (e) { showBrepGuidance(e); }               // §4 폴백 안내
}
```

### 3.2 서버 (convert-server/server.py) — `/convert/3dm-to-glb` 라우트 추가

기존 `/convert/3dm-to-mvr`와 동형. Rhino Compute(:8081)의 `Mesh.CreateFromBrep`로
메시 생성 → glTF로 직렬화 후 반환. *(server.py 미열람 — 다음 단계에서 실제 패치)*

## 4. 임시 수정 (Interim Fix) — 서버 없이도 '빈 화면'은 막는다

서버 미가동 환경을 위해, 실패한 Brep에 **바운딩박스 프록시 + 명확한 안내**.

```js
// 실패한 Brep을 BBox 프록시로 대체 (rhino3dm Brep.getBoundingBox 사용)
function brepBBoxProxy(brep) {
  const bb = brep.getBoundingBox?.();
  if (!bb) return null;
  const [mn, mx] = [bb.min, bb.max];
  // mn/mx로 박스 8정점·12삼각형 생성 → meshData 반환 (rhinoMeshToData 형식)
  return boxMeshData(mn, mx);
}
```

```js
// 파싱 종료 후, 전량 실패 시 사용자에게 실행 가능한 안내
function showBrepGuidance() {
  // 1) Rhino에서 객체 _Mesh 후 저장, 또는
  // 2) SaveAs 시 '렌더 메시 저장(Save render meshes)' 옵션 ON, 또는
  // 3) convert-server/start_convert_server.bat 실행 후 compute 모드 재시도
}
```

→ 효과: NURBS는 정밀하진 않아도 **공간 점유가 보여** 작업 연속성 확보 + 사용자가
무엇을 해야 하는지 즉시 인지.

## 5. 적용 방식 결정 필요 (→ 의뢰자 확인)

| 옵션 | 내용 | 비고 |
|------|------|------|
| **A. Compute 연결(권장)** | §3 — 정밀 메시, 기존 인프라 재사용 | `server.py`에 `/3dm-to-glb` 추가 필요(서버 코드 열람 후 패치) |
| **B. 임시 폴백** | §4 — 서버 없이 BBox+안내, 즉시 적용·실행 가능 | 정밀도 낮음(프록시) |
| **A+B 병행(최선)** | 서버 있으면 정밀, 없으면 프록시+안내 | 가장 견고 |

> 비파괴 원칙: 패치는 원본을 덮지 않고 **새 파일**(예:
> `index_patched_brep-compute_20260601.html`)로 생성·검수 후 교체.

## 6. 부수 발견 (참고)

- 다수의 `index.html` 사본이 드라이브에 산재(버전 혼재). **정본(canonical) 1개를
  지정**하고 나머지는 `_archive/`로 정리 권장.
- standalone의 브라우저 직접 AI 호출(localStorage 키)은 공유 시 키 노출 위험 →
  프로덕션은 서버 경유 권장(기존 인수인계 문서에도 명시).
