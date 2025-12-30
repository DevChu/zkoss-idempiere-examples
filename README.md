# Building the iDempiere ZK EE Components Example Plugin
This document provides a step-by-step guide on how to build the `org.idempiere.zkee.comps.example` plugin from the `zkoss-idempiere-examples` repository.

For general iDempiere plugin development guidelines, refer to the [iDempiere Wiki](https://wiki.idempiere.org/en/Developing_Plug-Ins_-_Get_your_Plug-In_running).

## Introduction

This repository demonstrates how to create an iDempiere plugin that uses ZK EE components.

We assume readers:
- Know the iDempiere basics
- Know the ZK framework basics

With this project you can:
- Build a ZK plugin with ZK EE components and install it into iDempiere
- Follow the example project to create your own plugin with ZK EE components

Building iDempiere plugins requires having the iDempiere core libraries available as a local p2 repository. This guide will walk you through the process of setting up the necessary dependencies and building the plugin.

## Prerequisites

Before you begin, ensure you have the following tools installed:

-   **Git:** For cloning the iDempiere repository.
-   **Maven:** For building the projects.
-   **Java Development Kit (JDK):** Version 17 or higher.
-   **iDempiere Runtime**: An active instance (e.g., [Official Docker Image](https://hub.docker.com/r/idempiereofficial/idempiere)).

## Step-by-step Guide

### 1. Clone iDempiere Core

The first step is to clone the main iDempiere project, which provides the core libraries needed to build the plugins.

```bash
# Clone version 12 of the iDempiere project
git clone --branch release-12 https://github.com/idempiere/idempiere.git idempiere
```

This will create a directory named `idempiere` containing the iDempiere source code.

### 2. Build iDempiere Core

This creates a local p2 repository at `idempiere/org.idempiere.p2/target/repository`.

```bash
cd idempiere
mvn clean install
```

### 3. Point the examples to your absolute core paths

The target platform files contain variable references that must be converted to absolute paths.

Edit these two files in a text editor:
- `idempiere/org.idempiere.p2.targetplatform/org.idempiere.p2.targetplatform.target`
- `idempiere/org.idempiere.p2.targetplatform/org.idempiere.p2.targetplatform.mirror.target`

Find the line containing:
```
${project_loc:org.idempiere.p2.targetplatform}
```

Replace it with the full absolute path to your directory, for example:
```
/Users/yourname/idempiere-plugin-dev/idempiere/org.idempiere.p2.targetplatform
```

### 4. Clone this Repository

Clone this repository into the same parent folder as iDempiere Core so both directories are siblings:

```bash
# From the parent directory (e.g., idempiere-plugin-dev)
git clone https://github.com/anthropics/zkoss-idempiere-examples.git
```

Your directory structure should look like:
```
idempiere-plugin-dev/
├── idempiere/
└── zkoss-idempiere-examples/
```

### 5. Add a ZK PE/EE fragment

Since we want the web UI to load ZK PE/EE widgets (e.g., from zkex and zkmax), use the fragment project `org.idempiere.zkee.comps.fragment`:

1) Build the fragment:
```bash
cd zkoss-idempiere-examples/org.idempiere.zkee.comps.fragment
mvn clean -U -DskipTests -am verify
```
   This runs the dependency-copy step and produces `org.idempiere.zkee.comps.fragment/target/org.idempiere.zkee.comps.fragment-<version>.jar`.
2) Install the fragment into your OSGi runtime (for example via Felix Web Console, or by placing the jar in the plugins directory) and restart the server so the host bundle (`org.adempiere.ui.zk`) resolves with the fragment on its classpath.
3) Confirm the fragment is active; the ZK PE/EE widgets (defined in the embedded `zkex`/`zkmax` lang-addons) should render without “widget class required” errors.

#### What is in `org.idempiere.zkee.comps.fragment`?

- Purpose: OSGi fragment that attaches both ZK PE and EE jars to `org.adempiere.ui.zk`, exposing lang-addons, widgets, and resources from zkex/zkmax plus `gson` for supporting components.
- Key files:
  - `META-INF/MANIFEST.MF`: `Fragment-Host: org.adempiere.ui.zk`, `Bundle-ClassPath: ., lib/zkex.jar, lib/zkmax.jar, lib/gson.jar`.
  - `build.properties`: includes `META-INF/`, `lib/zkex.jar`, `lib/zkmax.jar`, `lib/gson.jar` so they are packaged inside the fragment.
  - `pom.xml`: eclipse-plugin packaging; EE eval repository; dependency-copy execution to fetch `zkex`, `zkmax`, and `gson` into `lib/` (version-stripped).
  - `src/metainfo/zk.xml`: sets EE-specific library property (`org.zkoss.zkmax.au.IWBS.disable=true`).
  - `lib/zkex.jar`, `lib/gson.jar`, `lib/zkmax.jar`: ZK PE/EE binaries with `metainfo/zk/lang-addon.xml` and widget resources.
  - `target/`: built outputs (`org.idempiere.zkee.comps.fragment-<version>.jar`, generated manifest, p2 metadata).

### 6. Use ZK EE components in your own plugin (e.g., `org.idempiere.zkee.comps.example`)
 
1) Ensure the ZK EE fragment (`org.idempiere.zkee.comps.fragment`) is installed and active in the runtime; restart the server so `org.adempiere.ui.zk` resolves with the fragment on its classpath.
2) If your build cannot see the EE jar, add a dependency-copy step similar to the fragment (pulling the EE jar into `lib/`) or add the EE bundle to your target platform so Tycho can resolve it.
3) In ZUL, once the fragment is active, you can directly use EE components (e.g., `<timepicker .../>`) because the lang-addon from the fragment registers them. See `org.idempiere.zkee.comps.example/src/web/form.zul` for an example using EE widgets.
4) Build your plugin and redeploy; verify there are no “Widget class required…” errors and the EE components render.
```bash
cd zkoss-idempiere-examples/org.idempiere.zkee.comps.example
mvn clean verify
```
Artifacts are written to `target/`.

---

## Appendix: Why Fragment is Needed

### The Technical Reason

| Constraint | Explanation |
|------------|-------------|
| **OSGi classloaders** | Each OSGi bundle has its own classloader - bundles are isolated |
| **ZK's lang-addon.xml** | ZK discovers components via `metainfo/zk/lang-addon.xml` using the **host bundle's classloader** |
| **Fragment behavior** | A fragment shares the **same classloader** as its host bundle |

**Result**: To make `org.adempiere.ui.zk` "see" the ZK EE widgets (`zkex.jar`, `zkmax.jar`), those JARs must be on its classloader. A **fragment** is the only OSGi-compliant way to inject resources into another bundle's classloader without modifying the host.

### Architecture Flow

```
ZK EE widgets need to be discovered by ZK's classloader
    ↓
ZK runs inside org.adempiere.ui.zk bundle
    ↓
OSGi bundles have isolated classloaders
    ↓
Only a FRAGMENT can share the host's classloader
    ↓
Therefore: Fragment is required
```

### References

- [OSGi vogella blog](https://vogella.com/blog/osgi-bundles-fragments-dependencies/) - "A fragment is loaded in the same classloader as the host"
- [bnd Fragment-Host docs](https://bnd.bndtools.org/heads/fragment_host.html) - "A fragment is a bundle that is attached to a host bundle"
- [iDempiere Wiki - Make ZK WebApp OSGi](https://wiki.idempiere.org/en/Make_Zk_WebApp_OSGi) - iDempiere OSGi architecture