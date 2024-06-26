example modified from https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/


```yaml
containers:
  - name: argocd-repo-server
    env:
      - name: ARGOCD_REPO_SERVER_PLUGIN_TAR_EXCLUSIONS
        value: '*'
  - name: my-plugin
    command: [/var/run/argocd/argocd-cmp-server] # Entrypoint should be Argo CD lightweight CMP server i.e. argocd-cmp-server
    image: busybox # This can be off-the-shelf or custom-built image
    securityContext:
      runAsNonRoot: true
      runAsUser: 999
    volumeMounts:
      - mountPath: /var/run/argocd
        name: var-files
      - mountPath: /home/argocd/cmp-server/plugins
        name: plugins
      # Remove this volumeMount if you've chosen to bake the config file into the sidecar image.
      - mountPath: /home/argocd/cmp-server/config/plugin.yaml
        subPath: plugin.yaml
        name: my-plugin-config
      # Starting with v2.4, do NOT mount the same tmp volume as the repo-server container. The filesystem separation helps mitigate path traversal attacks.
      - mountPath: /tmp
        name: tmp # cmp-tmp <--- !!! I AM going to use the same tmp volume as the repo-server... !!!
volumes:
  - configMap:
      name: my-plugin-config
    name: my-plugin-config
  # - emptyDir: {}
  #   name: cmp-tmp
```


```yaml
# my-plugin-config / plugin.yaml

apiVersion: argoproj.io/v1alpha1
kind: ConfigManagementPlugin
metadata:
  name: my-plugin
spec:
  generate:
    # `cd`ing into the matched "/tmp/_argocd-repo/<uuid>".
    # running the stdout producer (`cat`) on "/tmp/_argocd-repo/<uuid>/$ARGOCD_APP_SOURCE_PATH".
    command:
      - sh
      - -c
      - |
        cd /tmp/_argocd-repo; chmod +r .; chmod +rx *
        cd "$(grep -l "url = $ARGOCD_APP_SOURCE_REPO_URL" */.git/config)/../../$ARGOCD_APP_SOURCE_PATH"

        cat *.yaml

    # the default value for `workingDir` is "/tmp/_cmp-server/<uuid>/$ARGOCD_APP_SOURCE_PATH" (no breaking changes).
    # the `workingDir` term was chosen to match https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#entrypoint (since we already have `command` and `args`...).

    # I'm setting `workingDir` to repo's root ("/tmp/_cmp-server/<uuid>").
    # because in my deployment, I exclude all the files, and the streamed TarGz is empty.
    # hence, exec/fork on "/tmp/_cmp-server/<uuid>/$ARGOCD_APP_SOURCE_PATH", but the root does exists.
    workingDir: .
```

the two small needed changes for this example to work:

https://github.com/argoproj/argo-cd/compare/master...marxus:argo-cd:cmp-stream-allow-to-exclude-all

https://github.com/argoproj/argo-cd/compare/master...marxus:cmp-plugin-init-generate-working-dir

---

some benchmarks:

assuming this application manifest
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo
spec:
  project: default
  source:
    repoURL: https://github.com/marxus/argo-cd
    path: yamls
    targetRevision: cmp-required-changes-demo
    plugin:
      name: my-plugin
  destination:
    namespace: demo
    server: 'https://kubernetes.default.svc'
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

Running 3 hard refreshes<br/>
With the changes (Exclude all)
```
time="2024-06-26T12:10:15Z" level=info msg="ArgoCD ConfigManagementPlugin Server is starting" built="2024-06-26T07:50:10Z" commit=3f344d54a4e0bbbb4313e1c19cfe1e544b162598 version=v2.11.3+3f344d5.dirty
time="2024-06-26T12:10:15Z" level=info msg="No discovery configuration is defined for plugin my-plugin. To use this plugin, specify \"my-plugin\" in the Application's spec.source.plugin.name field."
time="2024-06-26T12:10:15Z" level=info msg="argocd-cmp-server v2.11.3+3f344d5.dirty serving on /home/argocd/cmp-server/plugins/my-plugin.sock"
time="2024-06-26T12:12:12Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=MatchRepository grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:12:12Z" grpc.time_ms=0.39 span.kind=server system=grpc
time="2024-06-26T12:12:12Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=MatchRepository grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:12:12Z" grpc.time_ms=0.525 span.kind=server system=grpc
time="2024-06-26T12:12:12Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=GetParametersAnnouncement grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:12:12Z" grpc.time_ms=0.13 span.kind=server system=grpc
time="2024-06-26T12:12:12Z" level=info msg="Generating manifests with no request-level timeout"
time="2024-06-26T12:12:12Z" level=info msg="sh -c \"cd /tmp/_argocd-repo; chmod +r .; chmod +rx *\\ncd \\\"$(grep -l \\\"url = $ARGOCD_APP_SOURCE_REPO_URL\\\" */.git/config)/../../$ARGOCD_APP_SOURCE_PATH\\\"\\n\\ncat *.yaml\\n\"" dir=/tmp/_cmp_server/62f3b7d3-c887-4e15-b187-678cfd413b85 execID=26c5e
time="2024-06-26T12:12:12Z" level=info msg="Plugin command successfull" command="{[sh -c cd /tmp/_argocd-repo; chmod +r .; chmod +rx *\ncd \"$(grep -l \"url = $ARGOCD_APP_SOURCE_REPO_URL\" */.git/config)/../../$ARGOCD_APP_SOURCE_PATH\"\n\ncat *.yaml\n] [] .}" execID=26c5e stderr=
time="2024-06-26T12:12:12Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=GenerateManifest grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:12:12Z" grpc.time_ms=3.261 span.kind=server system=grpc
time="2024-06-26T12:13:54Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=MatchRepository grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:13:54Z" grpc.time_ms=0.136 span.kind=server system=grpc
time="2024-06-26T12:13:54Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=MatchRepository grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:13:54Z" grpc.time_ms=0.144 span.kind=server system=grpc
time="2024-06-26T12:13:54Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=GetParametersAnnouncement grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:13:54Z" grpc.time_ms=0.193 span.kind=server system=grpc
time="2024-06-26T12:13:54Z" level=info msg="Generating manifests with no request-level timeout"
time="2024-06-26T12:13:54Z" level=info msg="sh -c \"cd /tmp/_argocd-repo; chmod +r .; chmod +rx *\\ncd \\\"$(grep -l \\\"url = $ARGOCD_APP_SOURCE_REPO_URL\\\" */.git/config)/../../$ARGOCD_APP_SOURCE_PATH\\\"\\n\\ncat *.yaml\\n\"" dir=/tmp/_cmp_server/cfcb9a64-357f-46ea-8fc3-293f5d971592 execID=232b8
time="2024-06-26T12:13:54Z" level=info msg="Plugin command successfull" command="{[sh -c cd /tmp/_argocd-repo; chmod +r .; chmod +rx *\ncd \"$(grep -l \"url = $ARGOCD_APP_SOURCE_REPO_URL\" */.git/config)/../../$ARGOCD_APP_SOURCE_PATH\"\n\ncat *.yaml\n] [] .}" execID=232b8 stderr=
time="2024-06-26T12:13:54Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=GenerateManifest grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:13:54Z" grpc.time_ms=2.954 span.kind=server system=grpc
time="2024-06-26T12:14:07Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=MatchRepository grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:14:07Z" grpc.time_ms=0.156 span.kind=server system=grpc
time="2024-06-26T12:14:07Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=MatchRepository grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:14:07Z" grpc.time_ms=0.156 span.kind=server system=grpc
time="2024-06-26T12:14:07Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=GetParametersAnnouncement grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:14:07Z" grpc.time_ms=0.195 span.kind=server system=grpc
time="2024-06-26T12:14:07Z" level=info msg="Generating manifests with no request-level timeout"
time="2024-06-26T12:14:07Z" level=info msg="sh -c \"cd /tmp/_argocd-repo; chmod +r .; chmod +rx *\\ncd \\\"$(grep -l \\\"url = $ARGOCD_APP_SOURCE_REPO_URL\\\" */.git/config)/../../$ARGOCD_APP_SOURCE_PATH\\\"\\n\\ncat *.yaml\\n\"" dir=/tmp/_cmp_server/95710552-fadd-4936-b6ba-3acd69af1e2b execID=d1f27
time="2024-06-26T12:14:07Z" level=info msg="Plugin command successfull" command="{[sh -c cd /tmp/_argocd-repo; chmod +r .; chmod +rx *\ncd \"$(grep -l \"url = $ARGOCD_APP_SOURCE_REPO_URL\" */.git/config)/../../$ARGOCD_APP_SOURCE_PATH\"\n\ncat *.yaml\n] [] .}" execID=d1f27 stderr=
time="2024-06-26T12:14:07Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=GenerateManifest grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:14:07Z" grpc.time_ms=3.245 span.kind=server system=grpc
```
```
times (ms):
GenerateManifest: 3.245,  2.954,  3.261
MatchRepository: 0.144,  0.136,  0.525 * 2(runs twice for some strange reason, even when plugin.name is explicet!)
GetParametersAnnouncement: 0.13,  0.193, 0.195

SUM: less than 5ms per refresh.
```

Running 3 hard refreshes<br/>
Without the changes (Excluding only to obvious '.git/*', still having 10000 files on 10k each under `noise`)
```
time="2024-06-26T12:31:32Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=MatchRepository grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:31:28Z" grpc.time_ms=3469.258 span.kind=server system=grpc
time="2024-06-26T12:31:32Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=MatchRepository grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:31:28Z" grpc.time_ms=3523.429 span.kind=server system=grpc
time="2024-06-26T12:31:35Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=GetParametersAnnouncement grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:31:32Z" grpc.time_ms=3327.828 span.kind=server system=grpc
time="2024-06-26T12:31:35Z" level=info msg="Generating manifests with no request-level timeout"
time="2024-06-26T12:31:35Z" level=info msg="sh -c \"cat *.yaml\"" dir=/tmp/_cmp_server/bde06cf1-076e-496d-befa-4ad19873b05e/yamls execID=0fca7
time="2024-06-26T12:31:35Z" level=info msg="Plugin command successfull" command="{[sh -c cat *.yaml] [] }" execID=0fca7 stderr=
time="2024-06-26T12:31:35Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=GenerateManifest grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:31:32Z" grpc.time_ms=3363.284 span.kind=server system=grpc
time="2024-06-26T12:34:52Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=MatchRepository grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:34:48Z" grpc.time_ms=3307.216 span.kind=server system=grpc
time="2024-06-26T12:34:52Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=MatchRepository grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:34:48Z" grpc.time_ms=3389.208 span.kind=server system=grpc
time="2024-06-26T12:34:55Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=GetParametersAnnouncement grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:34:52Z" grpc.time_ms=3297.304 span.kind=server system=grpc
time="2024-06-26T12:34:55Z" level=info msg="Generating manifests with no request-level timeout"
time="2024-06-26T12:34:55Z" level=info msg="sh -c \"cat *.yaml\"" dir=/tmp/_cmp_server/2f8d8316-9f74-4b4e-afe1-51275402fa67/yamls execID=0445e
time="2024-06-26T12:34:55Z" level=info msg="Plugin command successfull" command="{[sh -c cat *.yaml] [] }" execID=0445e stderr=
time="2024-06-26T12:34:55Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=GenerateManifest grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:34:52Z" grpc.time_ms=3313.829 span.kind=server system=grpc
time="2024-06-26T12:35:44Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=MatchRepository grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:35:40Z" grpc.time_ms=3294.555 span.kind=server system=grpc
time="2024-06-26T12:35:44Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=MatchRepository grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:35:40Z" grpc.time_ms=3325.467 span.kind=server system=grpc
time="2024-06-26T12:35:47Z" level=info msg="Generating manifests with no request-level timeout"
time="2024-06-26T12:35:47Z" level=info msg="sh -c \"cat *.yaml\"" dir=/tmp/_cmp_server/771a045f-db3a-4965-9b73-ecef53334e35/yamls execID=1f012
time="2024-06-26T12:35:47Z" level=info msg="Plugin command successfull" command="{[sh -c cat *.yaml] [] }" execID=1f012 stderr=
time="2024-06-26T12:35:47Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=GenerateManifest grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:35:44Z" grpc.time_ms=3383.591 span.kind=server system=grpc
time="2024-06-26T12:35:47Z" level=info msg="finished streaming call with code OK" grpc.code=OK grpc.method=GetParametersAnnouncement grpc.service=plugin.ConfigManagementPluginService grpc.start_time="2024-06-26T12:35:44Z" grpc.time_ms=3381.248 span.kind=server system=grpc
```
```
times (ms):
GenerateManifest: 3383.591,  3313.829,  3363.284
MatchRepository: 3389.208,  3523.429,  3325.467  * 2 (runs twice for some strange reason, even when plugin.name is explicet!)
GetParametersAnnouncement: 3381.248,  3297.304, 3327.828

SUM: around 12500ms per refresh.
```






and thats only with 1 app. when you have even 20 (not even talking about 400) apps,  it's not 12500ms * N of apps, everything is running on the same time, so it's raises to 90000 ms or 200000 ms per app and we even hit the 3000000+  per app!
this results in a halted cpu for over half an hour or more. and usually we get new commits before it even finish handling the prior one.

---
---
To conclude:

both changes i offer are logical and legit non breaking changes that doesnt directly address the issue at all. they just give you some legit -simple- configuration options.<br/>

i might "exploit" them and hack my way using those configuration options with 2 little shell lines and a volume mount... but thats up to me on my server...<br/>

other people might find them useful for other applications or their own hacks<br/>

plugins as they are now are not usable in a monorepo TarGZ streaming hell.
