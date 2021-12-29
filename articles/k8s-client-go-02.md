---
title: "k8s client-goでkubectl applyする"
emoji: "📌"
type: "tech"
topics: ["kubernetes", "go"]
published: false
---

client-goを使ってるとkubectl ではこうだけどどうやるんだっけ？ってことが結構起きるので諸々メモっていく

今回はkubectl create/apply したい

```go
package main

import (
	"context"
	"fmt"

	rbacv1 "k8s.io/api/rbac/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
)

func listExactMatch(ctx context.Context) (*rbacv1.RoleList, error) {
	labelSelector := &metav1.LabelSelector{
		MatchLabels: map[string]string{"app.kubernetes.io/managed-by": "sho2010"},
	}

	opts := metav1.ListOptions{
		LabelSelector: metav1.FormatLabelSelector(labelSelector),
	}

	list, err := client.RbacV1().Roles(metav1.NamespaceAll).List(ctx, opts)
	if err != nil {
    panic(err)
	}

	return list, nil
}

func listExists(ctx context.Context) (*rbacv1.RoleList, error) {
	requirement := metav1.LabelSelectorRequirement{
		Key:      "app.kubernetes.io/managed-by",
		Operator: metav1.LabelSelectorOpExists,
		Values:   []string{},
	}

	labelSelector := &metav1.LabelSelector{
		MatchExpressions: []metav1.LabelSelectorRequirement{
			requirement,
		},
	}

	opts := metav1.ListOptions{
		LabelSelector: metav1.FormatLabelSelector(labelSelector),
	}

	list, err := client.RbacV1().Roles(metav1.NamespaceAll).List(ctx, opts)
	if err != nil {
    panic(err)
	}

	return list, nil
}

```

~kubectl使えば？~

### 参考資料

- https://kubernetes.io/ja/docs/concepts/overview/working-with-objects/labels/
