# Maven

This Task can be used to run a Maven build.

## Install the Task

```
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/master/maven/maven.yaml
```

## Parameters

- **GOALS**: Maven `goals` to be executed
- **MAVEN_MIRROR_URL**: Maven mirror url (to be inserted into ~/.m2/settings.xml)

## Workspaces

* **source**: `git`-type `PipelineResource` specifying the location of the source to build.

## Usage

This TaskRun runs a Maven build

### With Defaults

```
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: maven-run
spec:
  workspaces:
  - name: source
    persistentVolumeClaim:
      claimName: my-source
  taskRef:
    name: maven
```
---

### With Custom Params

```
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: maven-run
spec:
  workspaces:
  - name: source
    persistentVolumeClaim:
      claimName: my-source
  params:
  - name: MAVEN_MIRROR_URL
    value: "http://localhost:8080/bucketrepo/"
  taskRef:
    name: maven
```
---
### With Custom /.m2/settings.yaml

A user provided custom `settings.xml` can be used with the Maven Task. To do this we need to mount the `settings.xml` on the Maven Task.
Following steps demostrate the use of a ConfigMap to mount a custom `settings.xml`.

1. create configmap
```
oc create configmap `maven-settings-cm` --from-flie settings.xml
```

2. modify Maven Task (mount config map to `maven-settings` step in Task definition. also add `maven-settings-cm` to volumes).
```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: maven
spec:
  params:
  - name: GOALS
    description: The Maven goals to run
    type: array
    default:
    - "package"
  - name: MAVEN_MIRROR_URL
    description: The Maven bucketrepo- mirror
    type: string
    default: ""
  workspaces:
  - name: source
    persistentVolumeClaim:
      claimName: my-source
  steps:
    - name: mvn-settings
      image: registry.access.redhat.com/ubi8/ubi-minimal:latest
      workingDir: /.m2
      command:
        - '/bin/bash'
        - '-c'
      args:
      - |-
        [[ -f /.m2/settings.xml ]] && \
        echo 'using already existing /.m2/settings.xml' && \
        cat /.m2/settings.xml && exit 0

        [[ -n '$(params.MAVEN_MIRROR_URL)' ]] && \
        cat > /.m2/settings.xml <<EOF
        <settings>
          <mirrors>
            <mirror>
              <id>mirror.default</id>
              <name>mirror.default</name>
              <url>$(params.MAVEN_MIRROR_URL)</url>
              <mirrorOf>*</mirrorOf>
            </mirror>
          </mirrors>
        </settings>
        EOF

        [[ -f /.m2/settings.xml ]] && cat /.m2/settings.xml
        [[ -f /.m2/settings.xml ]] || echo skipping settings
      volumeMounts:
        - name: m2-repository
          mountPath: /.m2
        - name: maven-configmap
          mountPath: /.m2/settings.xml
          subPath: settings.xml
    - name: mvn-goals
      image: gcr.io/cloud-builders/mvn
      workingDir: $(workspaces.source.path)
      command:
        - /usr/bin/mvn
      args:
        - "$(params.GOALS)"
      volumeMounts:
        - name: m2-repository
          mountPath: /.m2
  volumes:
    - name: m2-repository
      emptyDir: {}
    - name: maven-configmap
      configMap:
        name: maven-settings-cm
```
3. create TaskRun
