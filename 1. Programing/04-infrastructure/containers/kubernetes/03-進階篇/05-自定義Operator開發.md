# 05-自定義 Operator 開發

> 深入掌握生產級 Operator 開發的高級技巧

---

## 📚 本章目標

- 掌握 Controller 調和循環的高級用法
- 學會使用 Finalizer 實現資源清理
- 理解 Owner Reference 與級聯刪除
- 掌握狀態管理與條件更新
- 學會實現高級功能（備份、恢復、自動擴縮容）

---

## 1. Controller 高級模式

### 1.1 多資源 Watch

```go
func (r *WebAppReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&appsv1.WebApp{}).              // 主資源
		Owns(&appsv1.Deployment{}).        // 擁有的資源
		Owns(&corev1.Service{}).
		Watches(
			&source.Kind{Type: &corev1.ConfigMap{}},    // 監控其他資源
			handler.EnqueueRequestsFromMapFunc(r.findWebAppsForConfigMap),
		).
		Complete(r)
}

func (r *WebAppReconciler) findWebAppsForConfigMap(obj client.Object) []reconcile.Request {
	// 找到所有使用此 ConfigMap 的 WebApp
	configMap := obj.(*corev1.ConfigMap)
	webAppList := &appsv1.WebAppList{}
	
	err := r.List(context.Background(), webAppList)
	if err != nil {
		return []reconcile.Request{}
	}
	
	requests := make([]reconcile.Request, 0)
	for _, webapp := range webAppList.Items {
		if webapp.Spec.ConfigMapName == configMap.Name {
			requests = append(requests, reconcile.Request{
				NamespacedName: types.NamespacedName{
					Name:      webapp.Name,
					Namespace: webapp.Namespace,
				},
			})
		}
	}
	
	return requests
}
```

---

### 1.2 Finalizer 實現資源清理

```go
const webappFinalizer = "webapp.example.com/finalizer"

func (r *WebAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := log.FromContext(ctx)
	
	webapp := &appsv1.WebApp{}
	if err := r.Get(ctx, req.NamespacedName, webapp); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}
	
	// 檢查是否正在刪除
	if !webapp.ObjectMeta.DeletionTimestamp.IsZero() {
		// 資源正在刪除
		if controllerutil.ContainsFinalizer(webapp, webappFinalizer) {
			// 執行清理邏輯
			if err := r.finalizeWebApp(ctx, webapp); err != nil {
				return ctrl.Result{}, err
			}
			
			// 移除 finalizer
			controllerutil.RemoveFinalizer(webapp, webappFinalizer)
			if err := r.Update(ctx, webapp); err != nil {
				return ctrl.Result{}, err
			}
		}
		return ctrl.Result{}, nil
	}
	
	// 添加 finalizer
	if !controllerutil.ContainsFinalizer(webapp, webappFinalizer) {
		controllerutil.AddFinalizer(webapp, webappFinalizer)
		if err := r.Update(ctx, webapp); err != nil {
			return ctrl.Result{}, err
		}
	}
	
	// 正常調和邏輯
	return r.reconcileWebApp(ctx, webapp)
}

func (r *WebAppReconciler) finalizeWebApp(ctx context.Context, webapp *appsv1.WebApp) error {
	log := log.FromContext(ctx)
	log.Info("Finalizing WebApp", "name", webapp.Name)
	
	// 清理外部資源（例如：雲端資源、備份）
	// 1. 刪除備份
	if err := r.deleteBackups(ctx, webapp); err != nil {
		return err
	}
	
	// 2. 清理外部數據庫
	if err := r.cleanupExternalDB(ctx, webapp); err != nil {
		return err
	}
	
	log.Info("Successfully finalized WebApp")
	return nil
}
```

---

### 1.3 狀態條件管理

```go
// 定義條件類型
const (
	TypeReady     = "Ready"
	TypeDegraded  = "Degraded"
	TypeAvailable = "Available"
)

func (r *WebAppReconciler) updateStatus(ctx context.Context, webapp *appsv1.WebApp, deployment *appsv1.Deployment) error {
	// 更新副本數
	webapp.Status.Replicas = deployment.Status.Replicas
	webapp.Status.ReadyReplicas = deployment.Status.ReadyReplicas
	
	// 更新條件
	if deployment.Status.ReadyReplicas == *deployment.Spec.Replicas {
		meta.SetStatusCondition(&webapp.Status.Conditions, metav1.Condition{
			Type:               TypeReady,
			Status:             metav1.ConditionTrue,
			Reason:             "AllReplicasReady",
			Message:            "All replicas are ready",
			ObservedGeneration: webapp.Generation,
		})
		meta.SetStatusCondition(&webapp.Status.Conditions, metav1.Condition{
			Type:               TypeAvailable,
			Status:             metav1.ConditionTrue,
			Reason:             "MinimumReplicasAvailable",
			Message:            fmt.Sprintf("%d replicas available", webapp.Status.ReadyReplicas),
			ObservedGeneration: webapp.Generation,
		})
	} else {
		meta.SetStatusCondition(&webapp.Status.Conditions, metav1.Condition{
			Type:               TypeReady,
			Status:             metav1.ConditionFalse,
			Reason:             "ReplicasNotReady",
			Message:            fmt.Sprintf("%d/%d replicas ready", deployment.Status.ReadyReplicas, *deployment.Spec.Replicas),
			ObservedGeneration: webapp.Generation,
		})
	}
	
	// 更新狀態
	return r.Status().Update(ctx, webapp)
}
```

---

## 2. 實戰：Database Operator

完整示例：實現一個支持備份、恢復、自動擴容的 PostgreSQL Operator。

### 2.1 CRD 定義

```go
// api/v1/postgres_types.go
type PostgresSpec struct {
	Version  string           `json:"version"`
	Replicas int32            `json:"replicas"`
	Storage  PostgresStorage  `json:"storage"`
	Backup   *BackupConfig    `json:"backup,omitempty"`
}

type PostgresStorage struct {
	Size         string `json:"size"`
	StorageClass string `json:"storageClass,omitempty"`
}

type BackupConfig struct {
	Enabled  bool   `json:"enabled"`
	Schedule string `json:"schedule"`    // Cron 表達式
	Retention int   `json:"retention"`   // 保留天數
}

type PostgresStatus struct {
	Phase      string             `json:"phase,omitempty"`
	Replicas   int32              `json:"replicas,omitempty"`
	LastBackup *metav1.Time       `json:"lastBackup,omitempty"`
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}
```

### 2.2 備份邏輯

```go
func (r *PostgresReconciler) reconcileBackup(ctx context.Context, pg *databasev1.Postgres) error {
	if pg.Spec.Backup == nil || !pg.Spec.Backup.Enabled {
		return nil
	}
	
	// 創建 CronJob 進行定時備份
	cronJob := &batchv1.CronJob{
		ObjectMeta: metav1.ObjectMeta{
			Name:      fmt.Sprintf("%s-backup", pg.Name),
			Namespace: pg.Namespace,
		},
		Spec: batchv1.CronJobSpec{
			Schedule: pg.Spec.Backup.Schedule,
			JobTemplate: batchv1.JobTemplateSpec{
				Spec: batchv1.JobSpec{
					Template: corev1.PodTemplateSpec{
						Spec: corev1.PodSpec{
							RestartPolicy: corev1.RestartPolicyOnFailure,
							Containers: []corev1.Container{
								{
									Name:  "backup",
									Image: fmt.Sprintf("postgres:%s", pg.Spec.Version),
									Command: []string{
										"/bin/bash",
										"-c",
										fmt.Sprintf(`
											TIMESTAMP=$(date +%%Y%%m%%d-%%H%%M%%S)
											pg_dump -h %s -U postgres mydb > /backup/backup-$TIMESTAMP.sql
											
											# 上傳到 S3
											aws s3 cp /backup/backup-$TIMESTAMP.sql s3://my-backups/
											
											# 刪除舊備份
											find /backup -name "backup-*.sql" -mtime +%d -delete
										`, pg.Name, pg.Spec.Backup.Retention),
									},
									Env: []corev1.EnvVar{
										{
											Name: "PGPASSWORD",
											ValueFrom: &corev1.EnvVarSource{
												SecretKeyRef: &corev1.SecretKeySelector{
													LocalObjectReference: corev1.LocalObjectReference{
														Name: fmt.Sprintf("%s-credentials", pg.Name),
													},
													Key: "password",
												},
											},
										},
									},
									VolumeMounts: []corev1.VolumeMount{
										{
											Name:      "backup",
											MountPath: "/backup",
										},
									},
								},
							},
							Volumes: []corev1.Volume{
								{
									Name: "backup",
									VolumeSource: corev1.VolumeSource{
										EmptyDir: &corev1.EmptyDirVolumeSource{},
									},
								},
							},
						},
					},
				},
			},
		},
	}
	
	// 設置 Owner Reference
	if err := controllerutil.SetControllerReference(pg, cronJob, r.Scheme); err != nil {
		return err
	}
	
	// 創建或更新
	if err := r.Create(ctx, cronJob); err != nil {
		if !errors.IsAlreadyExists(err) {
			return err
		}
	}
	
	return nil
}
```

---

## 3. 測試 Operator

### 3.1 單元測試

```go
// controllers/webapp_controller_test.go
package controllers

import (
	"context"
	"testing"
	
	. "github.com/onsi/ginkgo/v2"
	. "github.com/onsi/gomega"
	
	appsv1 "k8s.io/api/apps/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	
	webappv1 "github.com/myorg/webapp-operator/api/v1"
)

var _ = Describe("WebApp Controller", func() {
	Context("When reconciling a resource", func() {
		const resourceName = "test-webapp"
		
		ctx := context.Background()
		
		typeNamespacedName := types.NamespacedName{
			Name:      resourceName,
			Namespace: "default",
		}
		webapp := &webappv1.WebApp{}
		
		BeforeEach(func() {
			By("creating the custom resource for the Kind WebApp")
			err := k8sClient.Get(ctx, typeNamespacedName, webapp)
			if err != nil && errors.IsNotFound(err) {
				resource := &webappv1.WebApp{
					ObjectMeta: metav1.ObjectMeta{
						Name:      resourceName,
						Namespace: "default",
					},
					Spec: webappv1.WebAppSpec{
						Size:  3,
						Image: "nginx:1.27",
						Port:  80,
					},
				}
				Expect(k8sClient.Create(ctx, resource)).To(Succeed())
			}
		})
		
		AfterEach(func() {
			resource := &webappv1.WebApp{}
			err := k8sClient.Get(ctx, typeNamespacedName, resource)
			Expect(err).NotTo(HaveOccurred())
			
			By("Cleanup the specific resource instance WebApp")
			Expect(k8sClient.Delete(ctx, resource)).To(Succeed())
		})
		
		It("should successfully reconcile the resource", func() {
			By("Checking if Deployment was created")
			Eventually(func() error {
				deployment := &appsv1.Deployment{}
				return k8sClient.Get(ctx, typeNamespacedName, deployment)
			}).Should(Succeed())
			
			By("Checking Deployment replicas")
			deployment := &appsv1.Deployment{}
			Expect(k8sClient.Get(ctx, typeNamespacedName, deployment)).To(Succeed())
			Expect(*deployment.Spec.Replicas).To(Equal(int32(3)))
		})
	})
})
```

---

## 4. 打包與發布

### 4.1 使用 Helm Chart 打包

```bash
# 生成 Helm Chart
helm create webapp-operator

# 結構：
webapp-operator/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── rbac.yaml
    ├── crd.yaml
    └── service.yaml
```

### 4.2 發布到 OLM

```yaml
# ClusterServiceVersion (CSV)
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  name: webapp-operator.v0.1.0
spec:
  displayName: WebApp Operator
  description: Operator for managing web applications
  version: 0.1.0
  
  install:
    strategy: deployment
    spec:
      deployments:
      - name: webapp-operator
        spec:
          replicas: 1
          selector:
            matchLabels:
              name: webapp-operator
          template:
            spec:
              containers:
              - name: operator
                image: myregistry.io/webapp-operator:v0.1.0
  
  customresourcedefinitions:
    owned:
    - name: webapps.apps.example.com
      version: v1
      kind: WebApp
      displayName: WebApp
      description: Represents a web application
```

---

## 5. 小結

本章介紹了生產級 Operator 開發：

**高級模式：**
- ✅ 多資源 Watch
- ✅ Finalizer 資源清理
- ✅ 狀態條件管理

**實戰功能：**
- ✅ 自動備份與恢復
- ✅ 完整的測試覆蓋
- ✅ Helm Chart 打包
- ✅ OLM 發布

---

## 參考資料 (References)

1. [Kubebuilder Book - Advanced Topics](https://book.kubebuilder.io/cronjob-tutorial/cronjob-tutorial.html)
2. [Operator Best Practices](https://sdk.operatorframework.io/docs/best-practices/)
3. [OLM Integration](https://olm.operatorframework.io/)
