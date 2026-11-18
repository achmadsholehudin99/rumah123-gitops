# GitOps Migration — Phase 2 (infra manifests)

Paket ini memindahkan sisa workload simulasi Rumah123 yang masih dibuat
dengan `kubectl` ke dalam git, supaya isi cluster sama dengan isi repo
(prinsip GitOps: source of truth ada di git — sesuai koreksi mentor
18 Nov 2026).

## Cara pakai

Salin isi folder ini ke repo `rumah123-gitops`, di bawah
`clusters/k3s-rumah123-sim/` (jadi `clusters/k3s-rumah123-sim/infra/...`
dan file ini di `clusters/k3s-rumah123-sim/GITOPS-MIGRATION.md`), lalu
`git add -A && git commit && git push`. Flux (Kustomization
`clusters/k3s-rumah123-sim`) menerapkan semua YAML di path itu — tidak
perlu `kubectl apply` manual. Sebelum commit, pastikan Phase 1
(HelmRelease `search`, `property`, `static-pages`, `customerweb`) sudah
di repo, supaya tidak ada nama Deployment/Service yang diperebutkan
antara manifest di sini dan HelmRelease.

## Isi paket (`infra/`)

| File | Isi |
|---|---|
| `namespaces.yaml` | Namespace `rumah123portal-prod`, `core-prod`, `elasticsearch-prod`, `kafka` — semua dengan label `istio-injection: enabled` |
| `gateway-virtualservice.yaml` | Gateway `rumah123portal` + VirtualService `main-vs` (salinan apa adanya) |
| `nginx-router.yaml` | ConfigMap `nginx-router` (nginx.conf) + Deployment + Service `nginx-router` |
| `api.yaml` | Deployment + Service `api` (stub Python) |
| `core.yaml` | `graphql`, `consumersiteauth`, `redis`, `core-id-db`, `elasticsearch`, `kafka` — masing-masing Deployment + Service |
| `stub-configmaps.yaml` | ConfigMap `stub` (berisi `stub.py`) di `core-prod` DAN di `rumah123portal-prod` — ConfigMap itu namespaced, jadi dua namespace butuh salinannya masing-masing |
| `redis-portal.yaml` | Deployment + Service `redis-cache` dan `redis-session` di `rumah123portal-prod` — **direkonstruksi, lihat peringatan di bawah** |
| `cronjobs-etl.yaml` | ConfigMap `etl` (berisi `etl.py`) + 10 CronJob ETL di `core-prod` |

Yang sengaja TIDAK ada di paket ini: service `home` (sudah jadi
HelmRelease), `search`/`property`/`static-pages`/`customerweb` (Phase 1,
HelmRelease), dan 4 repository `searchsuggestion`/`propertyrepository`/
`userrepository`/`enquiry` (jadi HelmRelease di langkah berikutnya —
kalau ikut di sini, manifest dan Helm akan berebut kepemilikan nama
yang sama).

## Yang tetap DI LUAR git (pengecualian)

1. **Secret `basic-auth`** (namespace `rumah123portal-prod`, file
   htpasswd user `mentor`) — kredensial asli, TIDAK BOLEH di-commit.
   Deployment `nginx-router` di paket ini tetap mereferensikannya;
   Secret itu harus sudah ada di cluster (dibuat manual dengan
   `kubectl create secret`). Kalau Secret belum ada, pod nginx-router
   tidak bisa start.
2. **Cloudflare tunnel** (Deployment `cloudflared`, namespace `tunnel`)
   — URL trycloudflare.com sementara + kredensial tunnel; tetap di
   luar GitOps.

## Peringatan — baca sebelum apply

- **`redis-cache` / `redis-session` direkonstruksi dari analogi.**
  Tidak ada file YAML sumber untuk keduanya di mana pun (README hanya
  menyebut redis:7-alpine + nama Service-nya; tidak ada di
  `manifests/`). Manifest di `redis-portal.yaml` dimodelkan persis dari
  `redis` di core-prod (image, args, port 6379, readiness probe
  tcpSocket yang sama), hanya nama/namespace/selector yang diubah.
  Setelah apply, verifikasi pod-nya 2/2 dan `search`/`property`
  tetap jalan seperti sebelumnya.
- **Password Postgres `core-id-db` = `simulasi`** — nilai dummy untuk
  simulasi saja, bukan kredensial asli. Jangan dipakai di luar lab ini.
- **Label namespace** `istio-injection: enabled` untuk
  `elasticsearch-prod` dan `kafka` terkonfirmasi dari manifest asli;
  untuk `rumah123portal-prod` dan `core-prod` disimpulkan dari pod
  yang berjalan 2/2 (aplikasi + sidecar Envoy) dan dari anotasi
  opt-out `sidecar.istio.io/inject: 'false'` di CronJob ETL.

## Verifikasi setelah apply

```bash
flux get kustomizations
flux get helmreleases -A
kubectl get pods -n rumah123portal-prod   # harus 9 pod, semua 2/2 Running
kubectl get pods -n core-prod             # harus 8 pod, semua 2/2 Running
kubectl get cronjob -n core-prod          # 10 CronJob
curl -s -u mentor:<password> http://rumah123.local/
curl -s -u mentor:<password> http://rumah123.local/api/graphql
```

Yang diharapkan: homepage menjawab JSON `home` (200), dan
`/api/graphql` menjawab api 200 → graphql 200 → 5 upstream 200.
