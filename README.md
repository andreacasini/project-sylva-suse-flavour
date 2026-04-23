# 🗂️ Repository Structure

In this repository, you will find the following resources:

* 📄 **`dummy-capi-unit.md`** Describes how the `capi-controller` is overridden in favor of `rancher-turtles`.
* 📄 **`suse-flavored-sylva.md`** A summary of the project's progress as of April 2026.
* 📁 **`dummy-capi/`** Contains all necessary bits for the override to work. This needs to be referenced in your `values.yaml` when deploying a management cluster.
* 📁 **`suse-neuvector-init/`** Contains all necessary bits to allow for a custom version of NeuVector to be identical to SUSE Edge 3.5.
* 📁 **`mgmt-values-file/`** An example of the close-to-final configuration used to create a SUSE-flavored Sylva management cluster.