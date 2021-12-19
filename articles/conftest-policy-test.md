---
title: "RegoのPolicy test fictureにyamlを使う"
emoji: "📌"
type: "tech"
topics: ["rego", "conftest"]
published: false
---

# Summary

```rego
warn_imagePullPolicy_not_set[msg] {
	input.kind == "Deployment"
	not input.metadata.name == "overprovisioning"
	container := input.spec.template.spec.containers[_]
	not container.imagePullPolicy
	msg := sprintf("imagePullPolicy is not set to container %v of Deployment %v", [container.name, input.metadata.name])
}
```

```rego
test_imagePullPolicySet_not_set_on_deployment {
	manifest = `
    kind: Deployment
    metadata:
      name: some-app
    spec:
      template:
        spec:
          containers:
            - name: nginx
              image: nginx:latest
  `

	input := yaml.unmarshal(manifest)

	warn_imagePullPolicy_not_set["imagePullPolicy is not set to container nginx of Deployment some-app"] with input as input
}
```

