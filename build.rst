Building Ceph from Source on Local Environment, Creating a Container Image, and Deploying a Cluster
==================================================================================================
Overview
--------

This document describes how to:

1. Clone the Ceph source code
2. Optionally modify the source code
3. Build Ceph from source
4. Create a custom container image containing the built binaries
5. Deploy a Ceph cluster using the custom container image

This workflow is useful for:

- Testing development builds
- Running Ceph with custom patches
- Debugging issues using a locally built version
- Validating fixes before submitting upstream contributions

Prerequisites
-------------

Ensure the following tools are installed on the build host:

- Git
- Podman or Docker
- Internet access to download dependencies
- SSH connectivity between cluster nodes
- Basic familiarity with container runtimes

Cloning the Ceph Source Code
----------------------------

Clone the Ceph repository and initialize its submodules.

.. code-block:: bash

   git clone https://github.com/ceph/ceph.git
   cd ceph
   git submodule update --init --recursive --progress

The Ceph repository uses submodules for several dependencies.
Using the ``--recursive`` option ensures that all required components
are downloaded.

Optional: Creating a Development Branch
---------------------------------------

If you plan to modify the Ceph source code, it is recommended to create
a separate branch.

.. code-block:: bash

   git checkout -b wip-description-of-change

This step is optional and only required when testing code modifications.

Container-Based Build Workflow
------------------------------

Building Ceph inside a container provides a clean and reproducible
environment without installing build dependencies on the host system.

Step 1 — Clean old containers (optional)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Remove previously created containers if they exist.

.. code-block:: bash

   podman rm -a

This step ensures that older build containers do not interfere with
the current build process.

Step 2 — Start a build container
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Navigate to the Ceph source directory.

.. code-block:: bash

   cd ~/ceph

Start a build container and mount the Ceph source tree inside it.

.. code-block:: bash

   podman run -it \
     --name ceph-builder \
     -v $(pwd):/ceph:Z \
     quay.io/centos/centos:stream9 \
     bash

This command:

- launches a CentOS Stream 9 container
- mounts the Ceph source directory at ``/ceph``
- opens an interactive shell for building Ceph

Preparing the Build Environment
-------------------------------

Inside the container, install required packages.

.. code-block:: bash

   dnf install -y dnf-plugins-core
   dnf config-manager --set-enabled crb
   dnf install -y epel-release

Move to the Ceph source directory.

.. code-block:: bash

   cd /ceph

Install Ceph build dependencies.

.. code-block:: bash

   ./install-deps.sh

Install additional tools required during development.

.. code-block:: bash

   dnf install -y jq python3-routes doxygen graphviz

Cleaning Previous Builds
------------------------

If the repository already contains build artifacts, remove them.

.. code-block:: bash

   rm -rf build

This ensures that the build process starts from a clean state.

Configuring the Ceph Build
--------------------------

Run the Ceph CMake configuration script.

.. code-block:: bash

   ./do_cmake.sh \
      -DCMAKE_INSTALL_PREFIX=/usr \
      -DCMAKE_BUILD_TYPE=Release \
      -DWITH_DOCS=OFF \
      -DWITH_MGR_DASHBOARD=OFF \
      -DWITH_MGR_DASHBOARD_FRONTEND=OFF

These options configure Ceph with:

- optimized release build
- installation path under ``/usr``
- dashboard and documentation disabled to reduce build time

Building Ceph
-------------

Compile the source code using Ninja.

.. code-block:: bash

   ninja -C build -j$(nproc)

The ``-j$(nproc)`` option allows the build to run in parallel using
all available CPU cores.

Installing the Built Binaries
-----------------------------

Install the compiled binaries into the container filesystem.

.. code-block:: bash

   ninja -C build install
   ldconfig

The ``ldconfig`` command refreshes the shared library cache.

Creating Required Runtime Directories
-------------------------------------

Create directories required by Ceph daemons.

.. code-block:: bash

   mkdir -p /var/lib/ceph /etc/ceph /run/ceph

Create the Ceph system user and group if they do not exist.

.. code-block:: bash

   groupadd -g 167 ceph || true
   useradd -r -u 167 -g ceph ceph || true

Set appropriate permissions.

.. code-block:: bash

   chown -R ceph:ceph /var/lib/ceph

Verifying the Installation
--------------------------

Verify that the Ceph binaries are available.

.. code-block:: bash

   ceph --version
   ceph-authtool --version

Cleaning the Container Environment
----------------------------------

To reduce the size of the container image, remove cached packages
and build artifacts.

.. code-block:: bash

   dnf clean all
   rm -rf /var/cache/dnf
   rm -rf /ceph/build

Exit the container.

.. code-block:: bash

   exit

Creating a Container Image
--------------------------

Commit the running container as a new image.

.. code-block:: bash

   podman commit ceph-builder myceph/dev:latest

Export the container image to a tar archive.

.. code-block:: bash

   podman save -o ceph-daemon-latest.tar myceph/dev:latest

Note:
------
To access the `ceph-builder` container shell:

- If the container is **already running**, log in using:
  
  .. code-block:: bash

     podman exec -it ceph-builder bash

Deploying a Ceph Cluster Using the Custom Image
===============================================

Cluster Preparation
-------------------

Prepare cluster nodes where Ceph will run.

Example nodes:

- test-0 (bootstrap node)
- test-1
- test-2

Copy the Container Image
------------------------

Copy the saved container image to each node.

Example::

   scp ceph-daemon-latest.tar node:/home/user/

Load the Image
--------------

Load the container image on each node.

.. code-block:: bash

   podman load -i ~/ceph-daemon-latest.tar

Running a Local Container Registry
----------------------------------

Start a container registry on the bootstrap node.

Edit the registry configuration.

.. code-block:: bash

   mv /etc/containers/registries.conf /etc/containers/registries.bak
   vi /etc/containers/registries.conf

Add the following configuration:

::

   [registries.insecure]
   registries = ["test-0:5000"]

Create storage for the registry.

.. code-block:: bash

   mkdir -p /var/lib/registry
   chmod -R 777 /var/lib/registry

Start the registry container.

.. code-block:: bash

   podman pull quay.io/libpod/registry:2

   podman run -d \
      --name registry \
      -p 5000:5000 \
      --restart=always \
      -v /var/lib/registry:/var/lib/registry:Z \
      registry:2

Tagging the Image
-----------------

Tag the image for the local registry.

.. code-block:: bash

   podman tag myceph/dev:latest test-0:5000/myceph/dev:latest

Pushing the Image
-----------------

Push the image to the registry.

.. code-block:: bash

   podman push test-0:5000/myceph/dev:latest

Pulling the Image
-----------------

Ensure all nodes can access the image.

.. code-block:: bash

   podman pull test-0:5000/myceph/dev:latest

Bootstrapping the Ceph Cluster
------------------------------

Run the cephadm bootstrap command on the first node.

.. code-block:: bash

   cephadm --image test-0:5000/myceph/dev:latest \
      bootstrap \
      --mon-ip <test-0 IP> \
      --allow-overwrite \
      --allow-mismatched-release \
      --skip-dashboard

Configuring Host Resolution
---------------------------

If DNS is not configured, add host entries manually.

Example::

   10.0.0.10 test-0
   10.0.0.11 test-1
   10.0.0.12 test-2

Adding Hosts to the Cluster
---------------------------

Add additional nodes to the cluster from the bootstrap node.

.. code-block:: bash

   ceph orch host add test-1 <test-1 IP>
   ceph orch host add test-2 <test-2 IP>

Once hosts are added, Ceph orchestrator can deploy services such
as MONs, OSDs, and MGR daemons automatically.
