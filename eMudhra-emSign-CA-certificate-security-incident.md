# Security notice: Upgrade certifi, pip, and uv in response to CA certificate incident
https://discuss.python.org/t/information-for-python-users-regarding-emudhra-emsign-ca-certificate-security-incident/109576

- pip 26.2.2 requires Python 3.10 or newer. Older Python environments need separate remediation or migration.
- Windows and WSL have separate Python installations and certificate stores; update both.
- Update only installed, applicable tools and dependencies. If certifi is absent, there is no need to install it solely for this advisory. Other affected tools, such as conda and pixi, must be updated through their own supported mechanisms.

---

# windows

> **Windows shell:** Run the Windows commands in PowerShell, not Command Prompt (`cmd.exe`). The environment-variable syntax and virtual-environment activation commands shown here use PowerShell syntax.

## pip and certifi
```powershell
py -m pip install "pip @ https://files.pythonhosted.org/packages/3f/06/3e4420302363fbb71be3c7f35ce042f54bfca330e3e66b3a2e3c4cca579f/pip-26.2.2-py3-none-any.whl#sha256=fd7f4dfa8190e014013b9f08f3e973f4e4c26ffd179a204895533a7254813a70"
py -m pip install --upgrade pip
py -m pip install --upgrade certifi
```

## uv
For standalone uv installations that support self-updating, use the commands below. If uv was installed through pip, pipx, or another package manager, update it using that manager instead. Update the OS certificate store first.

```powershell
$env:UV_SYSTEM_CERTS = "true"
uv self update
```

## verify
```powershell
py -m pip --version
py -m pip show certifi
uv --version
```

required minimum versions
```
pip >= 26.2.2
certifi >= 2026.10.19
uv >= 0.13.4
```

## repeat for other python versions (if any)
```powershell
py -0p
```

```powershell
py -3.13 -m pip ...
```

--- 

# Linux / WSL / Docker for Ubuntu 22, 24, 26

Avoid using pip to install packages into Ubuntu's system-managed Python environment. Use virtual environments (venv or uv) for project dependencies. System Python packages should generally be managed through APT.

## update system certificates

https://ubuntu.com/security/notices/USN-8944-1

Update the operating system's CA certificate store to remove trust in the affected CA. This protects applications that rely on the system trust store. Applications with independent or bundled CA stores require separate updates.

```bash
sudo apt update
sudo apt install --only-upgrade ca-certificates
```

verify
```bash
lsb_release -a
dpkg-query -W -f='${Version}\n' ca-certificates
```

expected minimum version:
- 22.04 LTS jammy: 20260816ubuntu0.22.04.1
- 24.04 LTS noble: 20260816ubuntu0.24.04.1
- 26.04 LTS resolute: 20260816ubuntu0.26.04.1
- 26.10 stonking: 20260816ubuntu0.26.10.1

```bash
python3 -m pip --version
python3 -c "from pip._vendor import certifi; print(certifi.where())"
```

expected output for Ubuntu’s APT-managed pip:
```
pip <version> from /usr/lib/python3/dist-packages/pip (python 3.x)
/etc/ssl/certs/ca-certificates.crt
```

For Ubuntu’s APT-managed pip, verify that its CA configuration uses the updated system certificates. Do not apply the upstream pip version requirement directly to Ubuntu’s distribution-patched package. The path above identifies the bundle used by this certifi implementation; it does not verify every possible TLS configuration or custom CA override.

These checks apply only to the selected Python interpreter. Other Python installations and virtual environments may have independent copies of pip and certifi.

## upgrade uv
For standalone uv installations that support self-updating. For package-manager installations, use that manager instead.

```bash
UV_SYSTEM_CERTS=true uv self update
```

verify
```bash
uv --version
```

expected minimum version:
```
uv >= 0.13.4
```

---

# Virtual Environments Created with `python -m venv` and using `pip` as the package manager
Creating a new environment with `python -m venv` may seed an older pip from the Python installation. Verify or upgrade pip in new environments as well as existing ones.

windows:
```powershell
.\.venv\Scripts\Activate.ps1
```

linux:
```bash
source .venv/bin/activate
```

same for both windows and linux:

upgrade pip used in the virtual environment
```bash
python -m pip install "pip @ https://files.pythonhosted.org/packages/3f/06/3e4420302363fbb71be3c7f35ce042f54bfca330e3e66b3a2e3c4cca579f/pip-26.2.2-py3-none-any.whl#sha256=fd7f4dfa8190e014013b9f08f3e973f4e4c26ffd179a204895533a7254813a70"
python -m pip install --upgrade pip
```

Check whether certifi is installed, then upgrade it if applicable. Skip the install command if it is absent. Handle other affected tools using their own supported update mechanisms.
```bash
python -m pip show certifi
python -m pip install --upgrade certifi
```

verify
```bash
python -m pip --version
python -m pip show certifi
```

expected minimum versions:
```
pip >= 26.2.2
certifi >= 2026.10.19
```

---

# Virtual environments created and managed with `uv`
Updating your global `uv` installation fixes uv’s own certificate trust bundle across all environments managed by that updated executable. If an environment also contains pip (for example, after `uv venv --seed`), update that pip separately using the instructions above.

Check whether certifi is installed, then upgrade it if applicable. Skip the upgrade if it is absent. Handle other affected tools using their own supported update mechanisms.
```bash
uv pip show certifi
```

if no lockfile or pinned requirements file is used
```bash
uv pip install --upgrade certifi
```

if `uv.lock` is used
```bash
uv lock --upgrade-package certifi
uv sync
```

verify
```bash
uv pip show certifi
```

expected minimum version:
```
certifi >= 2026.10.19
```

If you use `uv.lock` and only run `uv pip install --upgrade certifi`, the next `uv sync` will restore the locked version, potentially downgrading certifi.

For projects that declare dependencies in `pyproject.toml` and use the `uv pip compile` workflow, regenerate and sync the compiled requirements:
```bash
uv pip compile pyproject.toml -o requirements.txt --upgrade-package certifi
uv pip sync requirements.txt
```

The first command reads dependencies from `pyproject.toml`, resolves exact versions (including transitive dependencies), and writes them to `requirements.txt`. `--upgrade-package certifi` allows certifi to upgrade rather than retaining its existing compiled pin, subject to dependency constraints. The second command installs the versions in `requirements.txt` into the target virtual environment and removes packages not listed there.

These commands do not create or update `uv.lock`. If your project uses `pyproject.toml` together with `uv.lock` and is managed with `uv sync`, use the `uv lock --upgrade-package certifi` and `uv sync` commands above instead.

Note: `requirements.txt` is the generated output in this workflow. If your project declares dependencies in `requirements.in` instead of `pyproject.toml`, substitute `requirements.in` as the input. For a manually maintained `requirements.txt`, update its affected pins through your normal dependency workflow rather than assuming it is generated. Preserve any project-specific compile and sync options, including hash generation, extras, and constraints.

If a requirements file, constraint file, or `pyproject.toml` pins certifi to an affected version, update that constraint first. This also applies to pip-managed projects. Commit updated dependency files and lockfiles to Git so rebuilds, teammates, and CI/CD retain the fix.

After updating dependencies, restart running applications and redeploy services so they reload certificate data. Check custom CA paths and overrides such as `REQUESTS_CA_BUNDLE`, `CURL_CA_BUNDLE`, and application-specific TLS settings separately.

---

# Docker Images

Docker containers have their own filesystem, packages, and CA certificate stores. Updating the host OS does not update existing container images.

For images based on Ubuntu, ensure the Dockerfile installs the patched ca-certificates package.

Rebuild the image without Docker layer caching and redeploy the containers:
```bash
docker build --pull --no-cache -t my-app:patched .
```

Also update any Python packages bundled in the image, including affected versions of pip, certifi, and uv.

Verify the versions inside the newly built image before deployment.

Updating a Docker base image does not automatically update vulnerable Python packages installed in later image layers.
