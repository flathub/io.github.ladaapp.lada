Run in project root directory:

## Release new version without dependency upgrades
You only need to adjust git commit of lada in `io.github.ladaapp.lada.yaml`.
Then create and merge a PR and given that the build pipeline ran through without issues wait a couple of hours before it becomes availalbe on Flathub.


## Release new version with dependency upgrades
In Flatpak land it seems to be common practise to build our dependencies for source and not rely on pre-build binaries.
This project is based on PyTorch/CUDA and there is no available SDK for flatpak for such projects so we have to rely on pre-build wheels from PyPi.
`req2flatpak` helps with this scenario and this is what we use to convert our pip dependencies to something flatpak can work with:


```shell
python -m venv .venv_build_flatpak
source .venv_build_flatpak/bin/activate
pip install pip-tools req2flatpak
pip-compile --extra gui,basicvsrpp -o flatpak/requirements.txt setup.py
req2flatpak --requirements-file flatpak/requirements.txt --target-platforms 312-x86_64 > flatpak/lada-pip-dependencies.json.tmp
````

MMCV is only published as source distribution on PyPi. We need the CUDA wheel.
We cannot build it via flatpak as there is no Flatpak SDK providing CUDA and Nvidia doesn't seem to provide an installation method which we could use to install it flatpak ourselves.
The MMCV project host their own Package Index with CUDA capable wheels but `req2flatpak` does only support official PyPi package index using their JSON API.
At the time of writing the MMCV project only offer binary wheels for Torch <= 2.4.x and as we want to use latest Torch we need to build it ourselves.
We also cannot easily downgrade to Torch 2.4.x as the PyTorch project only supports latest Torch versions on PyPi and all previous only via their own Package Index. Which we cannot use req2flatpak doesn't support it...
What a mess, so only option for now seems to be build the MMCV CUDA-capable binary wheel on our locally on our host machine and upload somewhere so that flathub buildbot is able to access it.
Then we need to replace the MMCV download source to this location.
Let's get to work:

## Build MMCV wheel
> Instructions adjusted from https://mmcv.readthedocs.io/en/latest/get_started/build.html

Make sure to install correct versions (should match flatpak GNOME runtime) -> Currently, this would be Torch 2.5.x, CUDA 12.x, Python 3.12.x

Install CUDA e.g. via your distribution package manager

Then clone mmcv repo and build the wheel
```shell
python -m venv .venv_build_mmcv
source .venv_build_mmcv/bin/activate
git clone https://github.com/open-mmlab/mmcv.git
cd mmcv
pip install torch # latest torch version build for CUDA, same version lada should be using
pip install setuptools wheel
pip install -r requirements/optional.txt
python setup.py bdist_wheel
```

Now add this file to lada flathub repo (root dir) (only needed if either Python (via Gnome Runtime), PyTorch, or MMCV itself was upgraded)
> Would be great not to commit this binary file but looks like we cannot create releases in flathub repo to attach files and uploading it to some 3rd-party file hoster probably isn't the best idea either....
```
mmcv/dist/mmcv-2.2.0-cp312-cp312-linux_x86_64.whl
```

Let's adjust the `req2flatpak` output and point to this file.

```shell
jq '.sources |= [.[] |select(.url == "https://files.pythonhosted.org/packages/e9/a2/57a733e7e84985a8a0e3101dfb8170fc9db92435c16afad253069ae3f9df/mmcv-2.2.0.tar.gz") = {type: "file", path: "mmcv-2.2.0-cp312-cp312-linux_x86_64.whl", sha256: "0c5fa8f302f99f6b64f11c767ceb504def84ef44849fc83378a99906463ac68e", "only-arches": [ "x86_64"]}]' flatpak/lada-pip-dependencies.json.tmp > flatpak/lada-pip-dependencies.json
```

Make sure the ULR and checksum are correct. You can get the checksum via:
```shell
sha256sum mmcv/dist/mmcv-2.2.0-cp312-cp312-linux_x86_64.whl
```

Now we have everything ready to build the flatpak.
Commit `lada-pip-dependencies.json` and create a PR and let Flathub buildbot do its thing.
For bigger changes you may want to build the flatpak yourself locally first to test and debug issues.

```shell
flatpak remote-add --if-not-exists --user flathub https://dl.flathub.org/repo/flathub.flatpakrepo
flatpak install --user -y flathub org.flatpak.Builder
mkdir flatpak
flatpak run org.flatpak.Builder --force-clean --sandbox --user --state-dir=flatpak/state --repo=flatpak/repo --install --install-deps-from=flathub --mirror-screenshots-url=https://dl.flathub.org/media/ flatpak/build  io.github.ladaapp.lada.yaml
```

