# Learning Garden — Linux Systems Learning Journal

A hands-on, bottom-up journal for learning Linux systems internals, published as a
[Just the Docs](https://just-the-docs.com/) site on GitHub Pages:

**https://moeinsoltani.github.io/learning-garden/**

It hosts **two independent learning tracks**, each a phased curriculum of
self-contained lessons. Every lesson follows the same shape: a mental model, the
mechanics ("why"), a lab with exact commands and expected output, and checkpoint
questions with hidden model answers.

## Tracks

### 🌐 Linux Networking
Network namespaces, virtual interfaces, bridges & VLANs, overlays, routing, NAT,
firewalling with nftables, traffic control, eBPF/XDP, and container/Kubernetes
networking. **47 lessons.**

→ [`_networking/learning-plan.md`](_networking/learning-plan.md)

### 🖥️ Linux Virtualization (QEMU / KVM)
CPU & hardware virtualization, KVM internals, QEMU, CPU/memory tuning, storage,
virtio, device passthrough (VFIO), libvirt, cloud images, live migration,
performance, security, and microVMs. **61 lessons.**

→ [`_virtualization/learning-plan.md`](_virtualization/learning-plan.md)

## Repository layout

```
index.md                  # landing page / track chooser
_config.yml               # Jekyll + Just the Docs config; defines both collections
_networking/
  learning-plan.md        # networking curriculum
  lessons/                # phase-NN-*.md parents + lesson-NN-*.md
_virtualization/
  learning-plan.md        # virtualization curriculum
  lessons/                # phase-NN-*.md parents + lesson-NN-*.md
CLAUDE.md                 # authoring conventions for the lessons
```

Each track is a Jekyll collection rendered as its own navigation section. A page's
URL is `/<track>/lessons/<file>.html`.

## Running the site locally

Requires Ruby with Bundler/Jekyll.

```bash
# install Jekyll + the remote-theme plugins
gem install jekyll bundler jekyll-remote-theme jekyll-seo-tag

# serve with the production baseurl so relative links resolve
jekyll serve --baseurl /learning-garden
# then open http://localhost:4000/learning-garden/
```

The published site builds automatically from the `main` branch via GitHub Pages.

## Authoring lessons

Lesson structure, front matter, callouts, and reference-link conventions are
documented in [`CLAUDE.md`](CLAUDE.md). Always consult the relevant track's
`learning-plan.md` before adding a lesson — it defines the topic and goals for
every lesson in order.
