# -*- mode: Python -*-

envsubst_cmd = "./hack/tools/bin/envsubst"
clusterctl_cmd = "./bin/clusterctl"
kubectl_cmd = "kubectl"

if str(local("command -v " + kubectl_cmd + " || true", quiet = True)) == "":
    fail("Required command '" + kubectl_cmd + "' not found in PATH")

load("ext://uibutton", "cmd_button", "location", "text_input")

# set defaults
version_settings(True, ">=0.22.2")

settings = {
    "deploy_cert_manager": True,
    "enable_providers": ["docker"],
    "kind_cluster_name": os.getenv("CAPI_KIND_CLUSTER_NAME", "capi-test"),
    "debug": {},
}

# global settings
tilt_file = "./tilt-settings.yaml" if os.path.exists("./tilt-settings.yaml") else "./tilt-settings.json"
settings.update(read_yaml(
    tilt_file,
    default = {},
))

os.putenv("CAPI_KIND_CLUSTER_NAME", settings.get("kind_cluster_name"))

allow_k8s_contexts(settings.get("allowed_contexts"))

os_name = str(local("go env GOOS")).rstrip("\n")
os_arch = str(local("go env GOARCH")).rstrip("\n")

if settings.get("trigger_mode") == "manual":
    trigger_mode(TRIGGER_MODE_MANUAL)

if settings.get("default_registry") != "":
    default_registry(settings.get("default_registry"))

always_enable_providers = ["core"]

providers = {
    "core": {
        "context": ".",
        "image": "gcr.io/k8s-staging-cluster-api/cluster-api-controller",
        "live_reload_deps": [
            "main.go",
            "go.mod",
            "go.sum",
            "api",
            "cmd",
            "controllers",
            "errors",
            "exp",
            "feature",
            "internal",
            "third_party",
            "util",
            "webhooks",
        ],
        "label": "CAPI",
    },
    "kubeadm-bootstrap": {
        "context": "bootstrap/kubeadm",
        "image": "gcr.io/k8s-staging-cluster-api/kubeadm-bootstrap-controller",
        "live_reload_deps": [
            "main.go",
            "api",
            "controllers",
            "internal",
            "types",
            "../../go.mod",
            "../../go.sum",
        ],
        "label": "CABPK",
    },
    "kubeadm-control-plane": {
        "context": "controlplane/kubeadm",
        "image": "gcr.io/k8s-staging-cluster-api/kubeadm-control-plane-controller",
        "live_reload_deps": [
            "main.go",
            "api",
            "controllers",
            "internal",
            "../../go.mod",
            "../../go.sum",
        ],
        "label": "KCP",
    },
    "docker": {
        "context": "test/infrastructure/docker",
        "image": "gcr.io/k8s-staging-cluster-api/capd-manager",
        "live_reload_deps": [
            "main.go",
            "../../go.mod",
            "../../go.sum",
            "../container",
            "api",
            "cloudinit",
            "controllers",
            "docker",
            "exp",
            "internal",
            "third_party",
        ],
        "additional_docker_helper_commands": "RUN curl -LO https://dl.k8s.io/release/v1.23.3/bin/linux/{ARCH}/kubectl && chmod +x ./kubectl && mv ./kubectl /usr/bin/kubectl".format(
            ARCH = os_arch,
        ),
        "additional_docker_build_commands": """
COPY --from=tilt-helper /usr/bin/kubectl /usr/bin/kubectl
""",
        "label": "CAPD",
    },
}

def ensure_clusterctl():
    local("make clusterctl")

# Reads a provider's tilt-provider.json file and merges it into the providers map.
# A list of dictionaries is also supported by enclosing it in brackets []
# An example file looks like this:
# {
#     "name": "aws",
#     "config": {
#         "image": "gcr.io/k8s-staging-cluster-api-aws/cluster-api-aws-controller",
#         "live_reload_deps": [
#             "main.go", "go.mod", "go.sum", "api", "cmd", "controllers", "pkg"
#         ]
#     }
# }
def load_provider_tiltfiles():
    provider_repos = settings.get("provider_repos", [])

    for repo in provider_repos:
        file = repo + "/tilt-provider.yaml" if os.path.exists(repo + "/tilt-provider.yaml") else repo + "/tilt-provider.json"
        if not os.path.exists(file):
            fail("Failed to load provider. No tilt-provider.{yaml|json} file found in " + repo)
        provider_details = read_yaml(file, default = {})
        if type(provider_details) != type([]):
            provider_details = [provider_details]
        for item in provider_details:
            provider_name = item["name"]
            provider_config = item["config"]
            if "context" in provider_config:
                provider_config["context"] = repo + "/" + provider_config["context"]
            else:
                provider_config["context"] = repo
            if "kustomize_config" not in provider_config:
                provider_config["kustomize_config"] = True
            if "go_main" not in provider_config:
                provider_config["go_main"] = "main.go"
            providers[provider_name] = provider_config

tilt_helper_dockerfile_header = """
# Tilt image
FROM golang:1.19.0 as tilt-helper
# Support live reloading with Tilt
RUN go install github.com/go-delve/delve/cmd/dlv@latest
RUN wget --output-document /restart.sh --quiet https://raw.githubusercontent.com/windmilleng/rerun-process-wrapper/master/restart.sh  && \
    wget --output-document /start.sh --quiet https://raw.githubusercontent.com/windmilleng/rerun-process-wrapper/master/start.sh && \
    chmod +x /start.sh && chmod +x /restart.sh && chmod +x /go/bin/dlv
"""

tilt_dockerfile_header = """
FROM gcr.io/distroless/base:debug as tilt
WORKDIR /
COPY --from=tilt-helper /start.sh .
COPY --from=tilt-helper /restart.sh .
COPY --from=tilt-helper /go/bin/dlv .
COPY manager .
"""

# Configures a provider by doing the following:
#
# 1. Enables a local_resource go build of the provider's manager binary
# 2. Configures a docker build for the provider, with live updating of the manager binary
# 3. Runs kustomize for the provider's config/default and applies it
def enable_provider(name, debug):
    p = providers.get(name)
    context = p.get("context")
    go_main = p.get("go_main", "main.go")
    label = p.get("label", name)
    debug_port = int(debug.get("port", 0))

    # Prefix each live reload dependency with context. For example, for if the context is
    # test/infra/docker and main.go is listed as a dep, the result is test/infra/docker/main.go. This adjustment is
    # needed so Tilt can watch the correct paths for changes.
    live_reload_deps = []
    for d in p.get("live_reload_deps", []):
        live_reload_deps.append(context + "/" + d)

    # Set up a local_resource build of the provider's manager binary. The provider is expected to have a main.go in
    # manager_build_path or the main.go must be provided via go_main option. The binary is written to .tiltbuild/bin/manager.
    # TODO @randomvariable: Race detector mode only currently works on x86-64 Linux.
    # Need to switch to building inside Docker when architecture is mismatched
    race_detector_enabled = debug.get("race_detector", False)
    if race_detector_enabled:
        if os_name != "linux" or os_arch != "amd64":
            fail("race_detector is only supported on Linux x86-64")
        cgo_enabled = "1"
        build_options = "-race"
        ldflags = "-linkmode external -extldflags \"-static\""
    else:
        cgo_enabled = "0"
        build_options = ""
        ldflags = "-extldflags \"-static\""

    if debug_port != 0:
        # disable optimisations and include line numbers when debugging
        gcflags = "all=-N -l"
    else:
        gcflags = ""

    build_env = "CGO_ENABLED={cgo_enabled} GOOS=linux GOARCH={arch}".format(
        cgo_enabled = cgo_enabled,
        arch = os_arch,
    )
    build_cmd = "{build_env} go build {build_options} -gcflags '{gcflags}' -ldflags '{ldflags}' -o .tiltbuild/bin/manager {go_main}".format(
        build_env = build_env,
        build_options = build_options,
        gcflags = gcflags,
        go_main = go_main,
        ldflags = ldflags,
    )

    local_resource(
        label.lower() + "_binary",
        cmd = "cd {context};mkdir -p .tiltbuild/bin;{build_cmd}".format(
            context = context,
            build_cmd = build_cmd,
        ),
        deps = live_reload_deps,
        labels = [label, "ALL.binaries"],
    )

    additional_docker_helper_commands = p.get("additional_docker_helper_commands", "")
    additional_docker_build_commands = p.get("additional_docker_build_commands", "")

    dockerfile_contents = "\n".join([
        tilt_helper_dockerfile_header,
        additional_docker_helper_commands,
        tilt_dockerfile_header,
        additional_docker_build_commands,
    ])

    port_forwards = []
    links = []

    if debug_port != 0:
        port_forwards.append(port_forward(debug_port, 30000))

    metrics_port = int(debug.get("metrics_port", 0))
    profiler_port = int(debug.get("profiler_port", 0))
    if metrics_port != 0:
        port_forwards.append(port_forward(metrics_port, 8080))
        links.append(link("http://localhost:" + str(metrics_port) + "/metrics", "metrics"))

    if profiler_port != 0:
        port_forwards.append(port_forward(profiler_port, 6060))
        links.append(link("http://localhost:" + str(profiler_port) + "/debug/pprof", "profiler"))

    # Set up an image build for the provider. The live update configuration syncs the output from the local_resource
    # build into the container.
    docker_build(
        ref = p.get("image"),
        context = context + "/.tiltbuild/bin/",
        dockerfile_contents = dockerfile_contents,
        target = "tilt",
        only = "manager",
        live_update = [
            sync(context + "/.tiltbuild/bin/manager", "/manager"),
            run("sh /restart.sh"),
        ],
    )

    if p.get("kustomize_config", True):
        yaml = read_file("./.tiltbuild/yaml/{}.provider.yaml".format(name))
        k8s_yaml(yaml)
        objs = decode_yaml_stream(yaml)
        k8s_resource(
            workload = find_object_name(objs, "Deployment"),
            objects = [find_object_qualified_name(objs, "Provider")],
            new_name = label.lower() + "_controller",
            labels = [label, "ALL.controllers"],
            port_forwards = port_forwards,
            links = links,
            resource_deps = ["provider_crd"],
        )

def find_object_name(objs, kind):
    for o in objs:
        if o["kind"] == kind:
            return o["metadata"]["name"]
    return ""

def find_object_qualified_name(objs, kind):
    for o in objs:
        if o["kind"] == kind:
            return "{}:{}:{}".format(o["metadata"]["name"], kind, o["metadata"]["namespace"])
    return ""

# Users may define their own Tilt customizations in tilt.d. This directory is excluded from git and these files will
# not be checked in to version control.
def include_user_tilt_files():
    user_tiltfiles = listdir("tilt.d")
    for f in user_tiltfiles:
        include(f)

# Enable core cluster-api plus everything listed in 'enable_providers' in tilt-settings.json
def enable_providers():
    for name in get_providers():
        enable_provider(name, settings.get("debug").get(name, {}))

def get_providers():
    user_enable_providers = settings.get("enable_providers", [])
    return {k: "" for k in user_enable_providers + always_enable_providers}.keys()

def deploy_provider_crds():
    # NOTE: we are applying raw yaml for clusterctl resources (vs delegating this to clusterctl methods) because
    # it is required to control precedence between creating this CRDs and creating providers.
    k8s_yaml(read_file("./.tiltbuild/yaml/clusterctl.crd.yaml"))
    k8s_resource(
        objects = ["providers.clusterctl.cluster.x-k8s.io:CustomResourceDefinition:default"],
        new_name = "provider_crd",
    )

def deploy_observability():
    if "promtail" in settings.get("deploy_observability", []):
        k8s_yaml(read_file("./.tiltbuild/yaml/promtail.observability.yaml"), allow_duplicates = True)
        k8s_resource(workload = "promtail", extra_pod_selectors = [{"app": "promtail"}], labels = ["observability"], resource_deps = ["loki"])

    if "loki" in settings.get("deploy_observability", []):
        k8s_yaml(read_file("./.tiltbuild/yaml/loki.observability.yaml"), allow_duplicates = True)
        k8s_resource(workload = "loki", port_forwards = "3100", extra_pod_selectors = [{"app": "loki"}], labels = ["observability"])

        cmd_button(
            "loki:import logs",
            argv = ["sh", "-c", "cd ./hack/tools/log-push && go run ./main.go --log-path=$LOG_PATH"],
            resource = "loki",
            icon_name = "import_export",
            text = "Import logs",
            inputs = [
                text_input("LOG_PATH", label = "Log path, one of: GCS path, ProwJob URL or local folder"),
            ],
        )

    if "grafana" in settings.get("deploy_observability", []):
        k8s_yaml(read_file("./.tiltbuild/yaml/grafana.observability.yaml"), allow_duplicates = True)
        k8s_resource(workload = "grafana", port_forwards = "3001:3000", extra_pod_selectors = [{"app": "grafana"}], labels = ["observability"])

    if "prometheus" in settings.get("deploy_observability", []):
        k8s_yaml(read_file("./.tiltbuild/yaml/prometheus.observability.yaml"), allow_duplicates = True)
        k8s_resource(workload = "prometheus-server", new_name = "prometheus", port_forwards = "9090", extra_pod_selectors = [{"app": "prometheus"}], labels = ["observability"])

    if "visualizer" in settings.get("deploy_observability", []):
        k8s_yaml(read_file("./.tiltbuild/yaml/visualizer.observability.yaml"), allow_duplicates = True)
        k8s_resource(
            workload = "capi-visualizer",
            new_name = "visualizer",
            port_forwards = [port_forward(local_port = 8000, container_port = 8081, name = "View visualization")],
            labels = ["observability"],
        )

def prepare_all():
    tools_arg = "--tools kustomize,envsubst "
    tilt_settings_file_arg = "--tilt-settings-file " + tilt_file

    cmd = "make -B tilt-prepare && ./hack/tools/bin/tilt-prepare {tools_arg}{tilt_settings_file_arg}".format(
        tools_arg = tools_arg,
        tilt_settings_file_arg = tilt_settings_file_arg,
    )
    local(cmd, env = settings.get("kustomize_substitutions", {}))

# create cluster template resources from cluster-template files in the templates directory
def cluster_templates():
    substitutions = settings.get("kustomize_substitutions", {})

    # Ensure we have default values for a small set of well-known variables
    substitutions["NAMESPACE"] = substitutions.get("NAMESPACE", "default")
    substitutions["KUBERNETES_VERSION"] = substitutions.get("KUBERNETES_VERSION", "v1.24.0")
    substitutions["CONTROL_PLANE_MACHINE_COUNT"] = substitutions.get("CONTROL_PLANE_MACHINE_COUNT", "1")
    substitutions["WORKER_MACHINE_COUNT"] = substitutions.get("WORKER_MACHINE_COUNT", "3")

    # Note: this is a workaround to pass env variables to cmd buttons while this is not supported natively like in local_resource
    for name, value in substitutions.items():
        os.environ[name] = value

    template_dirs = settings.get("template_dirs", {
        "docker": ["./test/infrastructure/docker/templates"],
    })

    for provider, provider_dirs in template_dirs.items():
        p = providers.get(provider)
        label = p.get("label", provider)

        for template_dir in provider_dirs:
            template_list = [filename for filename in listdir(template_dir) if os.path.basename(filename).endswith("yaml")]
            for filename in template_list:
                deploy_templates(filename, label, substitutions)

def deploy_templates(filename, label, substitutions):
    # validate filename exists
    if not os.path.exists(filename):
        fail(filename + " not found")

    basename = os.path.basename(filename)
    if basename.endswith(".yaml"):
        if basename.startswith("clusterclass-"):
            template_name = basename.replace("clusterclass-", "").replace(".yaml", "")
            deploy_clusterclass(template_name, label, filename, substitutions)
        elif basename.startswith("cluster-template-"):
            clusterclass_name = basename.replace("cluster-template-", "").replace(".yaml", "")
            deploy_cluster_template(clusterclass_name, label, filename, substitutions)

def deploy_clusterclass(clusterclass_name, label, filename, substitutions):
    apply_clusterclass_cmd = "cat " + filename + " | " + envsubst_cmd + " | " + kubectl_cmd + " apply --namespace=$NAMESPACE -f - && echo \"ClusterClass created from\'" + filename + "\', don't forget to delete\n\""
    delete_clusterclass_cmd = kubectl_cmd + " --namespace=$NAMESPACE delete clusterclass " + clusterclass_name + ' --ignore-not-found=true; echo "\n"'

    local_resource(
        name = clusterclass_name,
        cmd = ["bash", "-c", apply_clusterclass_cmd],
        env = substitutions,
        auto_init = False,
        trigger_mode = TRIGGER_MODE_MANUAL,
        labels = [label + ".clusterclasses"],
    )

    cmd_button(
        clusterclass_name + ":apply",
        argv = ["bash", "-c", apply_clusterclass_cmd],
        resource = clusterclass_name,
        icon_name = "note_add",
        text = "Apply `" + clusterclass_name + "` ClusterClass",
        inputs = [
            text_input("NAMESPACE", default = substitutions.get("NAMESPACE")),
        ],
    )

    cmd_button(
        clusterclass_name + ":delete",
        argv = ["bash", "-c", delete_clusterclass_cmd],
        resource = clusterclass_name,
        icon_name = "delete_forever",
        text = "Delete `" + clusterclass_name + "` ClusterClass",
        inputs = [
            text_input("NAMESPACE", default = substitutions.get("NAMESPACE")),
        ],
    )

def deploy_cluster_template(template_name, label, filename, substitutions):
    apply_cluster_template_cmd = "CLUSTER_NAME=" + template_name + "-$RANDOM;" + clusterctl_cmd + " generate cluster $CLUSTER_NAME --from " + filename + " | " + kubectl_cmd + " apply -f - && echo \"Cluster '$CLUSTER_NAME' created, don't forget to delete\n\""
    delete_clusters_cmd = 'DELETED=$(echo "$(bash -c "' + kubectl_cmd + ' --namespace=$NAMESPACE get clusters -A --no-headers -o custom-columns=":metadata.name"")" | grep -E "^' + template_name + '-[[:digit:]]{1,5}$"); if [ -z "$DELETED" ]; then echo "Nothing to delete for cluster template ' + template_name + '"; else echo "Deleting clusters:\n$DELETED\n"; echo $DELETED | xargs -L1 ' + kubectl_cmd + ' delete cluster; fi; echo "\n"'

    local_resource(
        name = template_name,
        cmd = ["bash", "-c", apply_cluster_template_cmd],
        env = substitutions,
        auto_init = False,
        trigger_mode = TRIGGER_MODE_MANUAL,
        labels = [label + ".templates"],
    )

    cmd_button(
        template_name + ":apply",
        argv = ["bash", "-c", apply_cluster_template_cmd],
        resource = template_name,
        icon_name = "add_box",
        text = "Create `" + template_name + "` cluster",
        inputs = [
            text_input("NAMESPACE", default = substitutions.get("NAMESPACE")),
            text_input("KUBERNETES_VERSION", default = substitutions.get("KUBERNETES_VERSION")),
            text_input("CONTROL_PLANE_MACHINE_COUNT", default = substitutions.get("CONTROL_PLANE_MACHINE_COUNT")),
            text_input("WORKER_MACHINE_COUNT", default = substitutions.get("WORKER_MACHINE_COUNT")),
        ],
    )

    cmd_button(
        template_name + ":delete",
        argv = ["bash", "-c", delete_clusters_cmd],
        resource = template_name,
        icon_name = "delete_forever",
        text = "Delete `" + template_name + "` clusters",
        inputs = [
            text_input("NAMESPACE", default = substitutions.get("NAMESPACE")),
        ],
    )

    cmd_button(
        template_name + ":delete-all",
        argv = ["bash", "-c", kubectl_cmd + " delete clusters --all --wait=false"],
        resource = template_name,
        icon_name = "delete_sweep",
        text = "Delete all workload clusters",
    )

##############################
# Actual work happens here
##############################

ensure_clusterctl()

include_user_tilt_files()

load_provider_tiltfiles()

prepare_all()

deploy_provider_crds()

deploy_observability()

enable_providers()

cluster_templates()
