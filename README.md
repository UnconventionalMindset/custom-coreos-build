# Fedora Core OS Custom config

## Build

Create a script to execute COSA in your home dir named cosa.sh and make it executable:
```
#!/bin/bash
env | grep COREOS_ASSEMBLER
COREOS_ASSEMBLER_CONTAINER_LATEST="quay.io/coreos-assembler/coreos-assembler:latest"
if [[ -z ${COREOS_ASSEMBLER_CONTAINER} ]] && $(podman image exists ${COREOS_ASSEMBLER_CONTAINER_LATEST}); then
    cosa_build_date_str="$(podman inspect -f "{{.Created}}" ${COREOS_ASSEMBLER_CONTAINER_LATEST} | awk '{print $1}')"
    cosa_build_date="$(date -d ${cosa_build_date_str} +%s)"
    if [[ $(date +%s) -ge $((cosa_build_date + 60*60*24*7)) ]] ; then
        echo -e "\e[0;33m----" >&2
        echo "The COSA container image is more that a week old and likely outdated." >&2
        echo "You should pull the latest version with:" >&2
        echo "podman pull ${COREOS_ASSEMBLER_CONTAINER_LATEST}" >&2
        echo -e "----\e[0m" >&2
        sleep 10
    fi
fi
set -x
podman run --env COSA_NO_KVM=1 --rm -ti --security-opt label=disable --privileged                                    \
    --uidmap=1000:0:1 --uidmap=0:1:1000 --uidmap 1001:1001:64536                          \
    -v ${PWD}:/srv/ --device /dev/kvm --device /dev/fuse                                  \
    --tmpfs /tmp -v /var/tmp:/var/tmp --name cosa                                         \
    ${COREOS_ASSEMBLER_CONFIG_GIT:+-v $COREOS_ASSEMBLER_CONFIG_GIT:/srv/src/config/:ro}   \
    ${COREOS_ASSEMBLER_GIT:+-v $COREOS_ASSEMBLER_GIT/src/:/usr/lib/coreos-assembler/:ro}  \
    ${COREOS_ASSEMBLER_CONTAINER_RUNTIME_ARGS}                                            \
    ${COREOS_ASSEMBLER_CONTAINER:-$COREOS_ASSEMBLER_CONTAINER_LATEST} "$@"

rc=$?; set +x; exit $rc
```


Save cosa alias:
```
cat <<EOF >> $HOME/.bashrc
alias cosa="$HOME/cosa.sh"
EOF
```

Update the terminal:
```
source ~/.bashrc
```

Update the stored COSA image:
```
podman pull quay.io/coreos-assembler/coreos-assembler:latest
```

Create the ISO:
```
mkdir fcos
cd fcos
cosa init https://github.com/UnconventionalMindset/fedora-coreos-custom-config.git
cosa fetch && cosa build && cosa buildextend-metal && cosa buildextend-metal4k && cosa buildextend-live
```


## Update repo submodule
```
git submodule update --remote fedora-coreos-config
```
