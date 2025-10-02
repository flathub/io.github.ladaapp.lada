Run in project root directory:

## Release new version without dependency upgrades
You only need to adjust git commit of lada in `io.github.ladaapp.lada.yaml`.
Then create and merge a PR and given that the build pipeline ran through without issues wait a couple of hours before it becomes availalbe on Flathub.


## Release new version with dependency upgrades
In Flatpak land it seems to be common practise to build our dependencies for source and not rely on pre-build binaries.
This project is based on PyTorch/CUDA and there is no available SDK for flatpak for such projects so we have to rely on pre-build wheels from PyPi.
`req2flatpak` helps with this scenario and this is what we use to convert the pip dependencies (available as requirements-gui.txt in lada upstream) to something flatpak can work with:


```shell
git clone https://github.com/ladaapp/lada.git # you maybe want to check-out a specific tag/commit here
cd lada
python -m venv .venv_build_flatpak
source .venv_build_flatpak/bin/activate
pip install setuptools req2flatpak
req2flatpak --requirements-file packaging/requirements-gui.txt --target-platforms 313-x86_64 > lada-pip-dependencies.json
deactivate
````

Now we have everything ready to build the flatpak.
Commit `lada-pip-dependencies.json`,  create a PR and let Flathub buildbot do its thing.
For bigger changes you may want to build the flatpak yourself locally first to test and debug issues:

```shell
flatpak remote-add --if-not-exists --user flathub https://dl.flathub.org/repo/flathub.flatpakrepo
flatpak install --user -y flathub org.flatpak.Builder
mkdir flatpak
flatpak run org.flatpak.Builder --force-clean --sandbox --user --state-dir=flatpak/state --repo=flatpak/repo --install --install-deps-from=flathub --mirror-screenshots-url=https://dl.flathub.org/media/ flatpak/build  io.github.ladaapp.lada.yaml
```

