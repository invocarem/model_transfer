# How to use `model_transfer` (RDMA mode)

SSH and the link between spark1 and spark2 are assumed to work already. This doc is only **how to run this program**.

---

### 1. Install (once per machine)

On **spark1** and **spark2**:

```bash
cd ~/code/model_transfer    # or wherever you cloned this
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install torch
```

---

### 2. Configure `.env`

There are **two checked-in examples** (same values except rank and paths):

| File | Machine | Role |
|------|---------|------|
| **`env.master.example`** | spark1 (head) | Sender, `RANK=0` |
| **`env.worker.example`** | spark2 (worker) | Receiver, `RANK=1` |

On **spark1**, copy the master file to `.env`:

```bash
cp env.master.example .env
# edit .env: paths and MASTER_ADDR if needed
```

On **spark2**, copy the worker file to `.env`:

```bash
cp env.worker.example .env
# edit .env: paths and MASTER_ADDR if needed
```

`MODEL_TRANSFER_MASTER_ADDR` must be **spark1’s IP** on **both** machines. `MODEL_TRANSFER_SRC` must point at the **same** file tree on both (shared storage). `MODEL_TRANSFER_WORLD_SIZE` and `MODEL_TRANSFER_MASTER_PORT` must match.

Meaning of the variables:

| Variable | What it does |
|----------|----------------|
| `MODEL_TRANSFER_MODE` | `rdma` |
| `MODEL_TRANSFER_RANK` | `0` master, `1` worker |
| `MODEL_TRANSFER_WORLD_SIZE` | `2` on both |
| `MODEL_TRANSFER_MASTER_ADDR` | Spark1’s IP on both |
| `MODEL_TRANSFER_MASTER_PORT` | Same on both |
| `MODEL_TRANSFER_SRC` | Model folder (same tree visible on both hosts) |
| `MODEL_TRANSFER_DEST` | Master: local path ≠ `SRC` (skip logic). Worker: output folder on spark2 |
| `MODEL_TRANSFER_TORCH_BACKEND` | `gloo` (default) or `nccl` — see below |
| `MODEL_TRANSFER_INIT_TIMEOUT_SEC` | How long to wait for the other node (default `1800`) |

### PyTorch backend: you do **not** need NCCL for the default

The script sends bytes with PyTorch **distributed**. **NCCL is only for GPU tensors.** The old logic picked NCCL whenever CUDA was present but still used **CPU** tensors, which **does not work**.

- **`MODEL_TRANSFER_TORCH_BACKEND=gloo`** (default in `env.*.example`): CPU tensors, **no NCCL**, fine for two machines over regular TCP. Use this unless you know you want NCCL.

- **`MODEL_TRANSFER_TORCH_BACKEND=nccl`**: uses **GPU** tensors; requires **CUDA + working NCCL** on **both** spark1 and spark2, matching PyTorch/CUDA builds. Use the same Docker image (or same stack) on both nodes. If NCCL fails, set `NCCL_DEBUG=INFO` when you run Python to see errors. You may need `NCCL_SOCKET_IFNAME=eth0` (replace with your NIC name).

Docker: run **the same image** on master and worker, mount the repo and data, activate the env, then run the two commands as usual.

---

### 3. Run

1. **spark2** (worker), in the repo with venv activated:

   ```bash
   python model_transfer.py
   ```

2. **spark1** (master), right after:

   ```bash
   python model_transfer.py
   ```

Overrides: `python model_transfer.py --help`.

---

### 4. If something goes wrong

- **Nothing copies on the master:** On spark1, `MODEL_TRANSFER_DEST` equals `MODEL_TRANSFER_SRC` → use a different `DEST` on the master (see `env.master.example`).
- **Error or hang on startup:** Both processes must run; `MASTER_ADDR` is spark1’s IP on both; same port.
- **NCCL / distributed errors:** Try `MODEL_TRANSFER_TORCH_BACKEND=gloo` first. For `nccl`, ensure both nodes have GPU, matching containers, and check `NCCL_DEBUG=INFO`.
