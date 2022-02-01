---
title: Generating a Fully-Qualified and Hashed Python Requirements File
description: A discussion and script showing how to generate a fully-qualified and hashed requirements file using python.
tags:
  - Python
  - Pip
---

Since I [discovered how to look up all hashes for a python
package](/blog/finding-python-package-hashes-via-api/) I thought I'd
explore what it would look like to automate generating a requirements
file with all packages pinned to an exact version and set of hashes.
The goal would be to enable a similar workflow to the one outlined in
[my post about repeatable
builds](/blog/repeatable-python-builds-with-pip/), but with the added
benefits of [hash
checking](https://pip.pypa.io/en/stable/topics/repeatable-installs/#hash-checking).

Here is what I came up with (this code is MIT licensed, see
[gitlab](https://gitlab.com/pdebelak/dotfiles/-/blob/83082cd567f5edd4da90d2297246af9c42b98397/scripts/pip-hash-freeze)
for details):

{% highlight python %}
#!/usr/bin/env python3
"""
Freezes packages from a requirements.txt file and a python
environment (usually a virtualenv) into a fully frozen and hashed set
of packages.

    . my-venv/bin/activate
    pip-hash-freeze requirements.txt >frozen-requirements.txt
    pip install -r frozen-requirements.txt
"""
import argparse
import fileinput
import json
from typing import Generator, Iterable
from urllib.request import urlopen

import pkg_resources


class Requirement:
    """Represents a single package requirement."""

    def __init__(self, requirement: pkg_resources.DistInfoDistribution):
        self.name = requirement.project_name
        self.version = requirement.version

    def __str__(self) -> str:
        """
        A string representatation suitable for printing into a
        requirements file
        """
        return " \\\n    ".join([f"{self.name}=={self.version}", *self.hashes()])

    def __eq__(self, other) -> bool:
        """Equality check used for deduping"""
        return hash(self) == hash(other)

    def __hash__(self) -> int:
        """Make hashable so it can deduped using a set"""
        return hash((self.name, self.version))

    def hashes(self) -> Generator[str, None, None]:
        """
        Fetch sha256 hashes for this version of this package

        Returns them in a format suitable for printing into a
        requirements file.
        """
        with urlopen(f"https://pypi.org/pypi/{self.name}/{self.version}/json") as f:
            data = json.load(f)
        for release in data["releases"][self.version]:
            sha = release["digests"]["sha256"]
            yield f"--hash=sha256:{sha}"


def parser() -> argparse.ArgumentParser:
    """Create argument parser for help text"""
    p = argparse.ArgumentParser(
        description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter
    )
    p.add_argument(
        "requirements_files",
        nargs="*",
        help="One or more requirements files to freeze, "
        "defaults to stdin if no files are provided.",
    )
    return p


def resolve_requirements(
    requirements: Iterable[str],
) -> Generator[Requirement, None, None]:
    """Use pkg_resources to parse and resolve requirements"""
    for requirement in pkg_resources.working_set.resolve(
        pkg_resources.parse_requirements(requirements)
    ):
        yield Requirement(requirement)


def main():
    """
    Print sorted list of fully-qualified and -hashed requirements from
    given requirements.
    """
    parser().parse_args()
    for requirement in sorted(
        set(resolve_requirements(fileinput.input())), key=lambda req: req.name.lower()
    ):
        print(requirement)


if __name__ == "__main__":
    main()
{% endhighlight %}

The workflow for using this is very similar the one workflow in [my
post about repeatable
builds](/blog/repeatable-python-builds-with-pip/), but with a few
differences:

1. Instead of a constraints file meant to be passed to `pip install
   -c` it generates a requirements file meant to be used with `pip
   install -r`.

   The fact that this can't just generate a constraints file [seems to
   be a bug](https://github.com/pypa/pip/issues/9243), but for now
   this still gives us the benefits of separating requirements and
   constraints since we keep the separate file just listing our
   top-level dependencies.
2. Because it generates a requirements file instead of a constraints
   file, if you have separate `requirements.txt` and
   `requirements-dev.txt` files (to separate dev dependencies like
   `pytest` out from the dependencies your app needs at runtime) you
   need to generate separate frozen requirements files for each input
   requirements file.

Let's take a look at how you can use this:

{% highlight shell %}
python3 -m venv .venv
. .venv/bin/activate
echo 'requests
pandas' >requirements.txt
pip install -r requirements.txt
pip-hash-freeze requirements.txt >frozen-requirements.txt
{% endhighlight %}

At the time of writing, this will give you a `frozen-requirements.txt`
file looking like:

```
certifi==2021.10.8 \
    --hash=sha256:d62a0163eb4c2344ac042ab2bdf75399a71a2d8c7d47eac2e2ee91b9d6339569 \
    --hash=sha256:78884e7c1d4b00ce3cea67b44566851c4343c120abd683433ce934a68ea58872
charset-normalizer==2.0.11 \
    --hash=sha256:2842d8f5e82a1f6aa437380934d5e1cd4fcf2003b06fed6940769c164a480a45 \
    --hash=sha256:98398a9d69ee80548c762ba991a4728bfc3836768ed226b3945908d1a688371c
idna==3.3 \
    --hash=sha256:84d9dd047ffa80596e0f246e2eab0b391788b0503584e8945f2368256d2735ff \
    --hash=sha256:9d643ff0a55b762d5cdb124b8eaa99c66322e2157b69160bc32796e824360e6d
numpy==1.22.1 \
    --hash=sha256:3d62d6b0870b53799204515145935608cdeb4cebb95a26800b6750e48884cc5b \
    --hash=sha256:831f2df87bd3afdfc77829bc94bd997a7c212663889d56518359c827d7113b1f \
    --hash=sha256:8d1563060e77096367952fb44fca595f2b2f477156de389ce7c0ade3aef29e21 \
    --hash=sha256:69958735d5e01f7b38226a6c6e7187d72b7e4d42b6b496aca5860b611ca0c193 \
    --hash=sha256:45a7dfbf9ed8d68fd39763940591db7637cf8817c5bce1a44f7b56c97cbe211e \
    --hash=sha256:7e957ca8112c689b728037cea9c9567c27cf912741fabda9efc2c7d33d29dfa1 \
    --hash=sha256:800dfeaffb2219d49377da1371d710d7952c9533b57f3d51b15e61c4269a1b5b \
    --hash=sha256:65f5e257987601fdfc63f1d02fca4d1c44a2b85b802f03bd6abc2b0b14648dd2 \
    --hash=sha256:632e062569b0fe05654b15ef0e91a53c0a95d08ffe698b66f6ba0f927ad267c2 \
    --hash=sha256:0d245a2bf79188d3f361137608c3cd12ed79076badd743dc660750a9f3074f7c \
    --hash=sha256:26b4018a19d2ad9606ce9089f3d52206a41b23de5dfe8dc947d2ec49ce45d015 \
    --hash=sha256:f8ad59e6e341f38266f1549c7c2ec70ea0e3d1effb62a44e5c3dba41c55f0187 \
    --hash=sha256:60f19c61b589d44fbbab8ff126640ae712e163299c2dd422bfe4edc7ec51aa9b \
    --hash=sha256:2db01d9838a497ba2aa9a87515aeaf458f42351d72d4e7f3b8ddbd1eba9479f2 \
    --hash=sha256:bcd19dab43b852b03868796f533b5f5561e6c0e3048415e675bec8d2e9d286c1 \
    --hash=sha256:78bfbdf809fc236490e7e65715bbd98377b122f329457fffde206299e163e7f3 \
    --hash=sha256:c51124df17f012c3b757380782ae46eee85213a3215e51477e559739f57d9bf6 \
    --hash=sha256:88d54b7b516f0ca38a69590557814de2dd638d7d4ed04864826acaac5ebb8f01 \
    --hash=sha256:b5ec9a5eaf391761c61fd873363ef3560a3614e9b4ead17347e4deda4358bca4 \
    --hash=sha256:4ac4d7c9f8ea2a79d721ebfcce81705fc3cd61a10b731354f1049eb8c99521e8 \
    --hash=sha256:e60ef82c358ded965fdd3132b5738eade055f48067ac8a5a8ac75acc00cad31f \
    --hash=sha256:e348ccf5bc5235fc405ab19d53bec215bb373300e5523c7b476cc0da8a5e9973
pandas==1.4.0 \
    --hash=sha256:de62cf699122dcef175988f0714678e59c453dc234c5b47b7136bfd7641e3c8c \
    --hash=sha256:46a18572f3e1cb75db59d9461940e9ba7ee38967fa48dd58f4139197f6e32280 \
    --hash=sha256:73f7da2ccc38cc988b74e5400b430b7905db5f2c413ff215506bea034eaf832d \
    --hash=sha256:5229c95db3a907451dacebc551492db6f7d01743e49bbc862f4a6010c227d187 \
    --hash=sha256:fe454180ad31bbbe1e5d111b44443258730467f035e26b4e354655ab59405871 \
    --hash=sha256:784cca3f69cfd7f6bd7c7fdb44f2bbab17e6de55725e9ff36d6f382510dfefb5 \
    --hash=sha256:de8f8999864399529e8514a2e6bfe00fd161f0a667903655552ed12e583ae3cb \
    --hash=sha256:0f19504f2783526fb5b4de675ea69d68974e21c1624f4b92295d057a31d5ec5f \
    --hash=sha256:f045bb5c6bfaba536089573bf97d6b8ccc7159d951fe63904c395a5e486fbe14 \
    --hash=sha256:5280d057ddae06fe4a3cd6aa79040b8c205cd6dd21743004cf8635f39ed01712 \
    --hash=sha256:1f3b74335390dda49f5d5089fab71958812bf56f42aa27663ee4c16d19f4f1c5 \
    --hash=sha256:51e5da3802aaee1aa4254108ffaf1129a15fb3810b7ce8da1ec217c655b418f5 \
    --hash=sha256:f103a5cdcd66cb18882ccdc18a130c31c3cfe3529732e7f10a8ab3559164819c \
    --hash=sha256:4a8d5a200f8685e7ea562b2f022c77ab7cb82c1ca5b240e6965faa6f84e5c1e9 \
    --hash=sha256:b5af258c7b090cca7b742cf2bd67ad1919aa9e4e681007366c9edad2d6a3d42b \
    --hash=sha256:156aac90dd7b303bf0b91bae96c0503212777f86c731e41929c571125d26c8e9 \
    --hash=sha256:2dad075089e17a72391de33021ad93720aff258c3c4b68c78e1cafce7e447045 \
    --hash=sha256:1d59c958d6b8f96fdf850c7821571782168d5acfe75ccf78cd8d1ac15fb921df \
    --hash=sha256:55ec0e192eefa26d823fc25a1f213d6c304a3592915f368e360652994cdb8d9a \
    --hash=sha256:23c04dab11f3c6359cfa7afa83d3d054a8f8c283d773451184d98119ef54da97 \
    --hash=sha256:cdd76254c7f0a1583bd4e4781fb450d0ebf392e10d3f12e92c95575942e37df5
python-dateutil==2.8.2 \
    --hash=sha256:961d03dc3453ebbc59dbdea9e4e11c5651520a876d0f4db161e8674aae935da9 \
    --hash=sha256:0123cacc1627ae19ddf3c27a5de5bd67ee4586fbdd6440d9748f8abb483d3e86
pytz==2021.3 \
    --hash=sha256:3672058bc3453457b622aab7a1c3bfd5ab0bdae451512f6cf25f64ed37f5b87c \
    --hash=sha256:acad2d8b20a1af07d4e4c9d2e9285c5ed9104354062f275f3fcd88dcef4f1326
requests==2.27.1 \
    --hash=sha256:f22fa1e554c9ddfd16e6e41ac79759e17be9e492b3587efa038054674760e72d \
    --hash=sha256:68d7c56fd5a8999887728ef304a6d12edc7be74f1cfa47714fc8b414525c9a61
six==1.16.0 \
    --hash=sha256:8abb2f1d86890a2dfb989f9a77cfcfd3e47c2a354b01111771326f8aa26e0254 \
    --hash=sha256:1e61c37477a1626458e36f7b1d82aa5c9b094fa4802892072e49de9c60c4c926
urllib3==1.26.8 \
    --hash=sha256:000ca7f471a233c2251c6c7023ee85305721bfdf18621ebff4fd17a8653427ed \
    --hash=sha256:0e7c33d9a63e7ddfcb86780aac87befc2fbddf46c58dbb487e0855f7ceec283c
```

Then, you can do a repeatable build with the same versions of
everything and with hash-checking by running:

{% highlight shell %}
pip install -r frozen-requirements.txt
{% endhighlight %}

You can upgrade package versions later by running:

{% highlight shell %}
pip install --upgrade -r requirements.txt
pip-hash-freeze requirements.txt >frozen-requirements.txt
pip install -r frozen-requirements.txt
{% endhighlight %}

This does work great, but since you need a full python program to do
this I'm not sure it's really worth it in most cases. If you're really
worried about the supply chain attacks that hash checking prevents,
you're probably best off using
[pipenv](https://pypi.org/project/pipenv/) or
[poetry](https://python-poetry.org/) rather than this script. For now,
I'll most likely stick to the constraints-based approach when working
with others since that can be handled with `pip` alone.
