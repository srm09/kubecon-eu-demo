# -*- mode: Python -*-

default_build_engine = "docker"

os_name = str(local("go env GOOS")).rstrip("\n")
os_arch = str(local("go env GOARCH")).rstrip("\n")

settings = {
    "debug": {},
    "build_engine": default_build_engine,
}

services =  {
    "backend": {
        "context": ".",  # NOTE: this should be kept in sync with corresponding setting in tilt-prepare
        "image": "foo.com/backend",
        "live_reload_deps": [
            "main.go",
            "go.mod",
            "go.sum",
        ],
        "label": "backend",
    }
}

tilt_helper_dockerfile_header = """
# Tilt image
FROM golang:1.19.6 as tilt-helper
# Support live reloading with Tilt
RUN go install github.com/go-delve/delve/cmd/dlv@latest
RUN wget --output-document /restart.sh --quiet https://raw.githubusercontent.com/tilt-dev/rerun-process-wrapper/master/restart.sh  && \
    wget --output-document /start.sh --quiet https://raw.githubusercontent.com/tilt-dev/rerun-process-wrapper/master/start.sh && \
    chmod +x /start.sh && chmod +x /restart.sh && chmod +x /go/bin/dlv && \
    touch /process.txt && chmod 0777 /process.txt `# pre-create PID file to allow even non-root users to run the image`
"""

tilt_dockerfile_header = """
FROM gcr.io/distroless/base:debug as tilt
WORKDIR /
COPY --from=tilt-helper /process.txt .
COPY --from=tilt-helper /start.sh .
COPY --from=tilt-helper /restart.sh .
COPY --from=tilt-helper /go/bin/dlv .
COPY $binary_name .
"""


def build_go_binary(context, reload_deps, debug, go_main, binary_name, label):
    # Set up a local_resource build of a go binary. The target repo is expected to have a main.go in
    # the context path or the main.go must be provided via go_main option. The binary is written to .tiltbuild/bin/{$binary_name}.
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

    debug_port = int(debug.get("port", 0))
    if debug_port != 0:
        # disable optimisations and include line numbers when debugging
        gcflags = "all=-N -l"
    else:
        gcflags = ""

    build_env = "CGO_ENABLED={cgo_enabled} GOOS=linux GOARCH={arch}".format(
        cgo_enabled = cgo_enabled,
        arch = os_arch,
    )

    build_cmd = "{build_env} go build {build_options} -gcflags '{gcflags}' -ldflags '{ldflags}' -o .tiltbuild/bin/{binary_name} {go_main}".format(
        build_env = build_env,
        build_options = build_options,
        gcflags = gcflags,
        go_main = go_main,
        ldflags = ldflags,
        binary_name = binary_name,
    )

    # Prefix each live reload dependency with context. For example, for if the context is
    # test/infra/docker and main.go is listed as a dep, the result is test/infra/docker/main.go. This adjustment is
    # needed so Tilt can watch the correct paths for changes.
    live_reload_deps = []
    for d in reload_deps:
        live_reload_deps.append(context + "/" + d)
    local_resource(
        label.lower() + "_binary",
        cmd = "cd {context};mkdir -p .tiltbuild/bin;{build_cmd}".format(
            context = context,
            build_cmd = build_cmd,
        ),
        deps = live_reload_deps,
        labels = [label, "ALL.binaries"],
    )

def build_docker_image(image, context, binary_name):
    links = []

    dockerfile_contents = "\n".join([
        tilt_helper_dockerfile_header,
        tilt_dockerfile_header,
    ])

    docker_build(
        ref = image,
        context = context + "/.tiltbuild/bin/",
        dockerfile_contents = dockerfile_contents,
        build_args = {"binary_name": binary_name},
        target = "tilt",
        only = binary_name,
        live_update = [
            sync(context + "/.tiltbuild/bin/" + binary_name, "/" + binary_name),
            run("sh /restart.sh"),
        ],
    )

def enable_backend(debug):
    s = services.get("backend")
    label = s.get("label")

    build_go_binary(
        context = s.get("context"),
        reload_deps = s.get("live_reload_deps"),
        debug = debug,
        go_main = s.get("go_main", "main.go"),
        binary_name = "backend",
        label = label,
    )

    build_docker_image(
        image = s.get("image"),
        context = s.get("context"),
        binary_name = "backend",
    )

    yaml = read_file("./yaml/{}.yaml".format("backend"))
    k8s_yaml(yaml)
    objs = decode_yaml_stream(yaml)
    k8s_resource(
        workload = find_object_name(objs, "Deployment"),
        objects = [find_object_qualified_name(objs, "Provider")],
        # new_name = label.lower() + "_controller",
        labels = [label],
        resource_deps = ["provider_crd"] + s.get("resource_deps", []),
    )

def enable_database():
    label = "database"
    yaml = read_file("./yaml/{}.yaml".format("mysql"))
    k8s_yaml(yaml)
    objs = decode_yaml_stream(yaml)
    k8s_resource(
        workload = find_object_name(objs, "Deployment"),
        # objects = [find_object_qualified_name(objs, "Provider")],
        # new_name = label.lower() + "_controller",
        labels = [label],
        resource_deps = [],
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

def find_all_objects_names(objs):
    qualified_names = []
    for o in objs:
        if "namespace" in o["metadata"] and o["metadata"]["namespace"] != "":
            qualified_names = qualified_names + ["{}:{}:{}".format(o["metadata"]["name"], o["kind"], o["metadata"]["namespace"])]
        else:
            qualified_names = qualified_names + ["{}:{}".format(o["metadata"]["name"], o["kind"])]
    return qualified_names

# enable_backend(settings.get("debug").get("backend", {}))
enable_database()

