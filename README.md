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
    # the default value for `workingDir` is "/tmp/_cmp-server/<uuid>/$ARGOCD_APP_SOURCE_PATH" (no breaking changes).
    # the `workingDir` term was chosen to match https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#entrypoint (since we already have `command` and `args`...).

    # I'm setting `workingDir` to repo's root ("/tmp/_cmp-server/<uuid>").
    # because in my deployment, I exclude all the files, and the streamed TarGz is empty.
    # hence, exec/fork on "/tmp/_cmp-server/<uuid>/$ARGOCD_APP_SOURCE_PATH", but the root does exists.
    workingDir: .

    # `cd`ing into the matched "/tmp/_argocd/<uuid>".
    # running the stdout producer (`cat`) on "/tmp/_argocd/<uuid>/$ARGOCD_APP_SOURCE_PATH".
    command:
      - sh
      - -c
      - |
        cd /tmp/_argocd-repo; chmod +r .; chmod +rx *
        cd "$(grep -l "url = $ARGOCD_APP_SOURCE_REPO_URL" */.git/config)/../../$ARGOCD_APP_SOURCE_PATH"

        cat *.yaml
```

the two small needed changes for this example to work:

https://github.com/argoproj/argo-cd/compare/master...marxus:argo-cd:cmp-stream-allow-to-exclude-all

https://github.com/argoproj/argo-cd/compare/master...marxus:cmp-plugin-init-generate-working-dir

both changes are logical and legit non breaking changes.<br/>
i might `hack` my way using those configuration options. but thats up to me.<br/>
other people might find them useful for other applications
