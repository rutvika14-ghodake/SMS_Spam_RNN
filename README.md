# SMS Spam RNN — Deployment on Ubuntu 24.04 LTS

## Why Ubuntu 24.04

Ubuntu 24.04 LTS ("Noble Numbat") ships with **Python 3.12** as its default system Python. This matters because TensorFlow currently does **not** publish wheels for Python 3.14 (the default on newer releases like Ubuntu 26.04), which causes `pip install tensorflow` to fail with no matching distribution found.

On 24.04, none of that applies — the steps below work with no extra tooling (no `uv`, no `deadsnakes` PPA, no manual Python version installs).

## Prerequisites

- EC2 instance running **Ubuntu 24.04 LTS**, sized at least `t3.small` (2GB RAM) — TensorFlow's install/runtime footprint can OOM a `t2.micro`.
- Security group inbound rules: port 22 (SSH, your IP) and port 5000 (for direct access) or port 80 (if fronted with nginx).
- SSH access to the instance.

## Setup

### 1. Install system packages
```bash
sudo apt update
sudo apt install python3-pip python3-venv nginx git -y
```

### 2. Clone the repo
```bash
git clone https://github.com/rutvika14-ghodake/SMS_Spam_RNN.git
cd SMS_Spam_RNN
```

### 3. Create and activate a virtual environment
```bash
python3 -m venv venv
source venv/bin/activate
python3 --version   # should print 3.12.x
```

### 4. Install dependencies
```bash
pip install -r requirements.txt
```

### 5. Run the app (quick check)
```bash
python3 app.py
```
Expected output:
```
Model loaded successfully!
 * Running on http://0.0.0.0:5000
```
CUDA/`cuInit` warnings are expected and harmless — the instance has no GPU, so TensorFlow runs on CPU.

Test from a second terminal:
```bash
curl -X POST -d "sequence_input=Free entry in 2 a wkly comp to win FA Cup" http://localhost:5000/predict
```

Stop the dev server with `Ctrl+C` once confirmed.

### 6. Run it properly with gunicorn
```bash
gunicorn --bind 0.0.0.0:5000 --workers 2 app:app
```
Both workers should log `Model loaded successfully!`. Visit:
```
http://<instance-public-ip>:5000
```

## Quick Start (subsequent runs)
```bash
ssh -i your-key.pem ubuntu@<instance-ip>
cd ~/SMS_Spam_RNN
source venv/bin/activate
gunicorn --bind 0.0.0.0:5000 --workers 2 app:app
```

> This runs in the foreground — closing the terminal/SSH session stops the app. Wrap it in a `systemd` service if you want it to persist across reboots and disconnects.

## Known Issue: Unreliable Predictions on Real Text

The classifier can confidently misclassify real messages. An obvious spam example ("Free entry in 2 a wkly comp to win FA Cup...") was classified as `HAM` with a probability of `0.0053`.

**Cause:** `app.py`'s `preprocess_input()` expects pre-tokenized numeric sequences. For plain text with no digits, it falls back to encoding each character by raw character code (`ord(char) % 1000`), which bears no relation to the vocabulary the model was actually trained on. The repo includes `RNNmodel.pkl` (model weights/architecture) but **not** the original tokenizer used during training, so real text can't be correctly vectorized.

This doesn't block deployment, but predictions on natural-language input aren't meaningful until the correct tokenizer is restored or the model is retrained with a saved tokenizer included.

## Next Steps

- [ ] Wrap gunicorn in a `systemd` service for persistence across reboots/disconnects.
- [ ] Put `nginx` in front on port 80, with gunicorn bound to `127.0.0.1:5000` only.
- [ ] Add HTTPS via `certbot --nginx` once on a domain.
- [ ] Fix the tokenizer/preprocessing issue so predictions are meaningful.
- [ ] Pin `keras`/`tensorflow` versions in `requirements.txt` (model was pickled with Keras `3.13.2`).
