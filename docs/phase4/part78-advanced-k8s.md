# Part 78: Advanced Kubernetes with Go

## เป้าหมายของบทเรียน
- เข้าใจ Custom Resource Definitions (CRDs)
- สร้าง Kubernetes Operators ด้วย Go
- ใช้ controller-runtime framework
- เรียนรู้ kubebuilder
- สร้าง custom controllers
- ทำ admission webhooks
- ใช้ Kubernetes client-go

---

## 1. Custom Resource Definitions (CRDs)

CRDs ให้เราขยาย Kubernetes API ด้วย resource types ของเราเอง

```go
// crd/types.go
package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// WebAppSpec กำหนด spec ของ WebApp
type WebAppSpec struct {
    // Replicas จำนวน replica ที่ต้องการ
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=100
    Replicas int32 `json:"replicas"`
    
    // Image Docker image ที่จะใช้
    Image string `json:"image"`
    
    // Port port ที่ application ฟัง
    // +kubebuilder:default=8080
    Port int32 `json:"port,omitempty"`
    
    // Resources resource requests/limits
    Resources ResourceRequirements `json:"resources,omitempty"`
    
    // AutoScaling การตั้งค่า auto scaling
    AutoScaling *AutoScalingSpec `json:"autoScaling,omitempty"`
}

// ResourceRequirements resource requests/limits
type ResourceRequirements struct {
    Requests ResourceList `json:"requests,omitempty"`
    Limits   ResourceList `json:"limits,omitempty"`
}

// ResourceList รายการ resources
type ResourceList struct {
    CPU    string `json:"cpu,omitempty"`
    Memory string `json:"memory,omitempty"`
}

// AutoScalingSpec การตั้งค่า auto scaling
type AutoScalingSpec struct {
    MinReplicas int32 `json:"minReplicas"`
    MaxReplicas int32 `json:"maxReplicas"`
    // TargetCPUUtilizationPercentage เปอร์เซ็นต์ CPU ที่จะ scale
    TargetCPUUtilizationPercentage int32 `json:"targetCPUUtilizationPercentage"`
}

// WebAppStatus สถานะของ WebApp
type WebAppStatus struct {
    // ObservedGeneration generation ล่าสุดที่ controller เห็น
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`
    
    // ReadyReplicas จำนวน replica ที่ ready
    ReadyReplicas int32 `json:"readyReplicas"`
    
    // AvailableReplicas จำนวน replica ที่ available
    AvailableReplicas int32 `json:"availableReplicas"`
    
    // Conditions สถานะของ conditions
    Conditions []metav1.Condition `json:"conditions,omitempty"`
    
    // URL URL ของ application
    URL string `json:"url,omitempty"`
}

// WebApp เป็น Custom Resource ของเรา
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.spec.replicas`
// +kubebuilder:printcolumn:name="Ready",type=integer,JSONPath=`.status.readyReplicas`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`
type WebApp struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    
    Spec   WebAppSpec   `json:"spec,omitempty"`
    Status WebAppStatus `json:"status,omitempty"`
}

// WebAppList เก็บ list ของ WebApp
// +kubebuilder:object:root=true
type WebAppList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []WebApp `json:"items"`
}
```

### CRD YAML Definition

```yaml
# config/crd/webapp_crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webapps.apps.example.com
spec:
  group: apps.example.com
  names:
    kind: WebApp
    listKind: WebAppList
    plural: webapps
    singular: webapp
    shortNames:
    - wa
  scope: Namespaced
  versions:
  - name: v1alpha1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required:
            - replicas
            - image
            properties:
              replicas:
                type: integer
                minimum: 1
                maximum: 100
              image:
                type: string
              port:
                type: integer
                default: 8080
              autoScaling:
                type: object
                properties:
                  minReplicas:
                    type: integer
                  maxReplicas:
                    type: integer
                  targetCPUUtilizationPercentage:
                    type: integer
          status:
            type: object
            properties:
              readyReplicas:
                type: integer
              availableReplicas:
                type: integer
              url:
                type: string
    subresources:
      status: {}
    additionalPrinterColumns:
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Ready
      type: integer
      jsonPath: .status.readyReplicas
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```

---

## 2. Kubernetes Operator Pattern

```go
// operator/webapp_controller.go
package controllers

import (
    "context"
    "fmt"
    
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/types"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"
    
    appsv1alpha1 "github.com/example/webapp-operator/api/v1alpha1"
)

// WebAppReconciler reconciles WebApp objects
type WebAppReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=apps.example.com,resources=webapps,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=apps.example.com,resources=webapps/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete

// Reconcile implements the reconciliation loop
func (r *WebAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    
    // Fetch the WebApp instance
    webapp := &appsv1alpha1.WebApp{}
    if err := r.Get(ctx, req.NamespacedName, webapp); err != nil {
        if errors.IsNotFound(err) {
            // Resource deleted, nothing to do
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }
    
    logger.Info("Reconciling WebApp", "name", webapp.Name, "namespace", webapp.Namespace)
    
    // Reconcile Deployment
    if err := r.reconcileDeployment(ctx, webapp); err != nil {
        return ctrl.Result{}, fmt.Errorf("reconciling deployment: %w", err)
    }
    
    // Reconcile Service
    if err := r.reconcileService(ctx, webapp); err != nil {
        return ctrl.Result{}, fmt.Errorf("reconciling service: %w", err)
    }
    
    // Update status
    if err := r.updateStatus(ctx, webapp); err != nil {
        return ctrl.Result{}, fmt.Errorf("updating status: %w", err)
    }
    
    return ctrl.Result{}, nil
}

// reconcileDeployment สร้างหรืออัปเดต Deployment
func (r *WebAppReconciler) reconcileDeployment(ctx context.Context, webapp *appsv1alpha1.WebApp) error {
    desired := r.buildDeployment(webapp)
    
    existing := &appsv1.Deployment{}
    err := r.Get(ctx, types.NamespacedName{
        Name:      webapp.Name,
        Namespace: webapp.Namespace,
    }, existing)
    
    if errors.IsNotFound(err) {
        // สร้าง Deployment ใหม่
        log.FromContext(ctx).Info("Creating Deployment", "name", desired.Name)
        return r.Create(ctx, desired)
    }
    
    if err != nil {
        return err
    }
    
    // อัปเดต Deployment ที่มีอยู่
    existing.Spec = desired.Spec
    return r.Update(ctx, existing)
}

// buildDeployment สร้าง Deployment spec
func (r *WebAppReconciler) buildDeployment(webapp *appsv1alpha1.WebApp) *appsv1.Deployment {
    labels := map[string]string{
        "app":        webapp.Name,
        "managed-by": "webapp-operator",
    }
    
    replicas := webapp.Spec.Replicas
    
    deployment := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      webapp.Name,
            Namespace: webapp.Namespace,
            Labels:    labels,
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: &replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: labels,
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: labels,
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "app",
                            Image: webapp.Spec.Image,
                            Ports: []corev1.ContainerPort{
                                {
                                    ContainerPort: webapp.Spec.Port,
                                    Protocol:      corev1.ProtocolTCP,
                                },
                            },
                            ReadinessProbe: &corev1.Probe{
                                ProbeHandler: corev1.ProbeHandler{
                                    HTTPGet: &corev1.HTTPGetAction{
                                        Path: "/health",
                                        Port: intstr(webapp.Spec.Port),
                                    },
                                },
                                InitialDelaySeconds: 5,
                                PeriodSeconds:       10,
                            },
                            LivenessProbe: &corev1.Probe{
                                ProbeHandler: corev1.ProbeHandler{
                                    HTTPGet: &corev1.HTTPGetAction{
                                        Path: "/healthz",
                                        Port: intstr(webapp.Spec.Port),
                                    },
                                },
                                InitialDelaySeconds: 15,
                                PeriodSeconds:       20,
                            },
                        },
                    },
                },
            },
        },
    }
    
    // Set owner reference
    ctrl.SetControllerReference(webapp, deployment, r.Scheme)
    
    return deployment
}

// reconcileService สร้างหรืออัปเดต Service
func (r *WebAppReconciler) reconcileService(ctx context.Context, webapp *appsv1alpha1.WebApp) error {
    desired := r.buildService(webapp)
    
    existing := &corev1.Service{}
    err := r.Get(ctx, types.NamespacedName{
        Name:      webapp.Name,
        Namespace: webapp.Namespace,
    }, existing)
    
    if errors.IsNotFound(err) {
        return r.Create(ctx, desired)
    }
    
    if err != nil {
        return err
    }
    
    existing.Spec.Ports = desired.Spec.Ports
    return r.Update(ctx, existing)
}

// buildService สร้าง Service spec
func (r *WebAppReconciler) buildService(webapp *appsv1alpha1.WebApp) *corev1.Service {
    labels := map[string]string{
        "app":        webapp.Name,
        "managed-by": "webapp-operator",
    }
    
    svc := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      webapp.Name,
            Namespace: webapp.Namespace,
            Labels:    labels,
        },
        Spec: corev1.ServiceSpec{
            Selector: labels,
            Ports: []corev1.ServicePort{
                {
                    Port:       80,
                    TargetPort: intstr(webapp.Spec.Port),
                },
            },
        },
    }
    
    ctrl.SetControllerReference(webapp, svc, r.Scheme)
    return svc
}

// updateStatus อัปเดต status ของ WebApp
func (r *WebAppReconciler) updateStatus(ctx context.Context, webapp *appsv1alpha1.WebApp) error {
    deployment := &appsv1.Deployment{}
    if err := r.Get(ctx, types.NamespacedName{
        Name:      webapp.Name,
        Namespace: webapp.Namespace,
    }, deployment); err != nil {
        return err
    }
    
    webapp.Status.ReadyReplicas = deployment.Status.ReadyReplicas
    webapp.Status.AvailableReplicas = deployment.Status.AvailableReplicas
    webapp.Status.ObservedGeneration = webapp.Generation
    webapp.Status.URL = fmt.Sprintf("http://%s.%s.svc.cluster.local", webapp.Name, webapp.Namespace)
    
    return r.Status().Update(ctx, webapp)
}

// intstr helper function
func intstr(port int32) interface{} {
    return port // simplified
}

// SetupWithManager เพิ่ม controller เข้า manager
func (r *WebAppReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&appsv1alpha1.WebApp{}).
        Owns(&appsv1.Deployment{}).
        Owns(&corev1.Service{}).
        Complete(r)
}
```

---

## 3. controller-runtime Framework

```go
// main.go - Operator entry point
package main

import (
    "flag"
    "os"
    
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/runtime"
    utilruntime "k8s.io/apimachinery/pkg/util/runtime"
    clientgoscheme "k8s.io/client-go/kubernetes/scheme"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/healthz"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
    
    appsv1alpha1 "github.com/example/webapp-operator/api/v1alpha1"
    "github.com/example/webapp-operator/controllers"
)

var (
    scheme   = runtime.NewScheme()
    setupLog = ctrl.Log.WithName("setup")
)

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))
    utilruntime.Must(appsv1alpha1.AddToScheme(scheme))
    utilruntime.Must(appsv1.AddToScheme(scheme))
    utilruntime.Must(corev1.AddToScheme(scheme))
}

func main() {
    var metricsAddr string
    var enableLeaderElection bool
    var probeAddr string
    
    flag.StringVar(&metricsAddr, "metrics-bind-address", ":8080", "The address the metric endpoint binds to.")
    flag.StringVar(&probeAddr, "health-probe-bind-address", ":8081", "The address the probe endpoint binds to.")
    flag.BoolVar(&enableLeaderElection, "leader-elect", false, "Enable leader election for controller manager.")
    
    opts := zap.Options{Development: true}
    opts.BindFlags(flag.CommandLine)
    flag.Parse()
    
    ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))
    
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:                 scheme,
        MetricsBindAddress:     metricsAddr,
        Port:                   9443,
        HealthProbeBindAddress: probeAddr,
        LeaderElection:         enableLeaderElection,
        LeaderElectionID:       "webapp-operator",
    })
    if err != nil {
        setupLog.Error(err, "unable to start manager")
        os.Exit(1)
    }
    
    if err = (&controllers.WebAppReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create controller", "controller", "WebApp")
        os.Exit(1)
    }
    
    // Health checks
    if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up health check")
        os.Exit(1)
    }
    if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up ready check")
        os.Exit(1)
    }
    
    setupLog.Info("starting manager")
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        os.Exit(1)
    }
}
```

---

## 4. Admission Webhooks

```go
// webhooks/webapp_webhook.go
package webhooks

import (
    "context"
    "fmt"
    "net/http"
    
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/webhook"
    "sigs.k8s.io/controller-runtime/pkg/webhook/admission"
    
    appsv1alpha1 "github.com/example/webapp-operator/api/v1alpha1"
)

// WebAppValidator validates WebApp objects
// +kubebuilder:webhook:path=/validate-apps-example-com-v1alpha1-webapp,mutating=false,failurePolicy=fail,sideEffects=None,groups=apps.example.com,resources=webapps,verbs=create;update,versions=v1alpha1,name=vwebapp.kb.io,admissionReviewVersions=v1

type WebAppValidator struct{}

// SetupWebhookWithManager เพิ่ม webhook เข้า manager
func (v *WebAppValidator) SetupWebhookWithManager(mgr ctrl.Manager) error {
    return ctrl.NewWebhookManagedBy(mgr).
        For(&appsv1alpha1.WebApp{}).
        WithValidator(v).
        Complete()
}

// ValidateCreate validates WebApp creation
func (v *WebAppValidator) ValidateCreate(ctx context.Context, obj runtime.Object) error {
    webapp, ok := obj.(*appsv1alpha1.WebApp)
    if !ok {
        return fmt.Errorf("expected WebApp, got %T", obj)
    }
    return v.validateWebApp(webapp)
}

// ValidateUpdate validates WebApp update
func (v *WebAppValidator) ValidateUpdate(ctx context.Context, oldObj, newObj runtime.Object) error {
    webapp, ok := newObj.(*appsv1alpha1.WebApp)
    if !ok {
        return fmt.Errorf("expected WebApp, got %T", newObj)
    }
    return v.validateWebApp(webapp)
}

// ValidateDelete validates WebApp deletion
func (v *WebAppValidator) ValidateDelete(ctx context.Context, obj runtime.Object) error {
    return nil
}

// validateWebApp validates WebApp spec
func (v *WebAppValidator) validateWebApp(webapp *appsv1alpha1.WebApp) error {
    if webapp.Spec.Replicas < 1 {
        return fmt.Errorf("replicas must be at least 1, got %d", webapp.Spec.Replicas)
    }
    
    if webapp.Spec.Image == "" {
        return fmt.Errorf("image cannot be empty")
    }
    
    if webapp.Spec.AutoScaling != nil {
        as := webapp.Spec.AutoScaling
        if as.MinReplicas > as.MaxReplicas {
            return fmt.Errorf("minReplicas (%d) cannot be greater than maxReplicas (%d)",
                as.MinReplicas, as.MaxReplicas)
        }
        if webapp.Spec.Replicas < as.MinReplicas || webapp.Spec.Replicas > as.MaxReplicas {
            return fmt.Errorf("replicas (%d) must be between minReplicas (%d) and maxReplicas (%d)",
                webapp.Spec.Replicas, as.MinReplicas, as.MaxReplicas)
        }
    }
    
    return nil
}

// WebAppMutator mutates WebApp objects
// +kubebuilder:webhook:path=/mutate-apps-example-com-v1alpha1-webapp,mutating=true,failurePolicy=fail,sideEffects=None,groups=apps.example.com,resources=webapps,verbs=create;update,versions=v1alpha1,name=mwebapp.kb.io,admissionReviewVersions=v1

type WebAppMutator struct{}

// Default sets default values for WebApp
func (m *WebAppMutator) Default(ctx context.Context, obj runtime.Object) error {
    webapp, ok := obj.(*appsv1alpha1.WebApp)
    if !ok {
        return fmt.Errorf("expected WebApp, got %T", obj)
    }
    
    // ตั้งค่า default สำหรับ port
    if webapp.Spec.Port == 0 {
        webapp.Spec.Port = 8080
    }
    
    // เพิ่ม labels ถ้าไม่มี
    if webapp.Labels == nil {
        webapp.Labels = make(map[string]string)
    }
    webapp.Labels["managed-by"] = "webapp-operator"
    
    return nil
}

// Admission webhook handler แบบ low-level
type PodSidecarInjector struct {
    decoder *admission.Decoder
}

func (p *PodSidecarInjector) Handle(ctx context.Context, req admission.Request) admission.Response {
    // Decode the pod
    pod := &corev1.Pod{}
    if err := p.decoder.Decode(req, pod); err != nil {
        return admission.Errored(http.StatusBadRequest, err)
    }
    
    // ตรวจสอบว่าควร inject sidecar หรือไม่
    if _, ok := pod.Labels["inject-sidecar"]; !ok {
        return admission.Allowed("no injection needed")
    }
    
    // Inject sidecar container
    sidecar := corev1.Container{
        Name:  "logging-sidecar",
        Image: "fluentd:latest",
    }
    pod.Spec.Containers = append(pod.Spec.Containers, sidecar)
    
    // Marshal กลับเป็น JSON
    marshaledPod, err := json.Marshal(pod)
    if err != nil {
        return admission.Errored(http.StatusInternalServerError, err)
    }
    
    return admission.PatchResponseFromRaw(req.Object.Raw, marshaledPod)
}

// InjectDecoder implements webhook.AdmissionHandler
func (p *PodSidecarInjector) InjectDecoder(d *admission.Decoder) error {
    p.decoder = d
    return nil
}
```

---

## 5. Kubernetes client-go

```go
// clientgo/client.go
package main

import (
    "context"
    "fmt"
    "os"
    "path/filepath"
    
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/watch"
    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/tools/clientcmd"
    "k8s.io/client-go/util/homedir"
)

// K8sClient wraps kubernetes client
type K8sClient struct {
    clientset *kubernetes.Clientset
}

// NewK8sClient สร้าง client ใหม่
func NewK8sClient() (*K8sClient, error) {
    // หา kubeconfig
    kubeconfig := os.Getenv("KUBECONFIG")
    if kubeconfig == "" {
        if home := homedir.HomeDir(); home != "" {
            kubeconfig = filepath.Join(home, ".kube", "config")
        }
    }
    
    config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
    if err != nil {
        return nil, fmt.Errorf("building config: %w", err)
    }
    
    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        return nil, fmt.Errorf("creating clientset: %w", err)
    }
    
    return &K8sClient{clientset: clientset}, nil
}

// ListDeployments แสดง deployments ใน namespace
func (k *K8sClient) ListDeployments(ctx context.Context, namespace string) ([]appsv1.Deployment, error) {
    deployments, err := k.clientset.AppsV1().Deployments(namespace).List(ctx, metav1.ListOptions{})
    if err != nil {
        return nil, err
    }
    return deployments.Items, nil
}

// GetDeployment คืน deployment
func (k *K8sClient) GetDeployment(ctx context.Context, namespace, name string) (*appsv1.Deployment, error) {
    return k.clientset.AppsV1().Deployments(namespace).Get(ctx, name, metav1.GetOptions{})
}

// ScaleDeployment scale deployment
func (k *K8sClient) ScaleDeployment(ctx context.Context, namespace, name string, replicas int32) error {
    scale, err := k.clientset.AppsV1().Deployments(namespace).GetScale(ctx, name, metav1.GetOptions{})
    if err != nil {
        return err
    }
    
    scale.Spec.Replicas = replicas
    _, err = k.clientset.AppsV1().Deployments(namespace).UpdateScale(ctx, name, scale, metav1.UpdateOptions{})
    return err
}

// WatchPods watch pod events
func (k *K8sClient) WatchPods(ctx context.Context, namespace string) error {
    watcher, err := k.clientset.CoreV1().Pods(namespace).Watch(ctx, metav1.ListOptions{})
    if err != nil {
        return err
    }
    defer watcher.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return nil
        case event, ok := <-watcher.ResultChan():
            if !ok {
                return fmt.Errorf("watcher channel closed")
            }
            
            pod, ok := event.Object.(*corev1.Pod)
            if !ok {
                continue
            }
            
            switch event.Type {
            case watch.Added:
                fmt.Printf("Pod Added: %s/%s\n", pod.Namespace, pod.Name)
            case watch.Modified:
                fmt.Printf("Pod Modified: %s/%s (Phase: %s)\n", 
                    pod.Namespace, pod.Name, pod.Status.Phase)
            case watch.Deleted:
                fmt.Printf("Pod Deleted: %s/%s\n", pod.Namespace, pod.Name)
            }
        }
    }
}

// UseInformer ใช้ informer สำหรับ caching
func (k *K8sClient) UseInformer(ctx context.Context) {
    factory := informers.NewSharedInformerFactory(k.clientset, 0)
    
    podInformer := factory.Core().V1().Pods().Informer()
    
    podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            pod := obj.(*corev1.Pod)
            fmt.Printf("[Informer] Pod Added: %s\n", pod.Name)
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            pod := newObj.(*corev1.Pod)
            fmt.Printf("[Informer] Pod Updated: %s\n", pod.Name)
        },
        DeleteFunc: func(obj interface{}) {
            pod := obj.(*corev1.Pod)
            fmt.Printf("[Informer] Pod Deleted: %s\n", pod.Name)
        },
    })
    
    factory.Start(ctx.Done())
    factory.WaitForCacheSync(ctx.Done())
    
    <-ctx.Done()
}

func main() {
    ctx := context.Background()
    
    fmt.Println("=== Kubernetes client-go Demo ===")
    fmt.Println("Note: This requires a running Kubernetes cluster")
    fmt.Println("The following shows the API usage patterns:")
    
    // แสดง code patterns (ไม่ต้อง connect จริง)
    fmt.Println(`
// 1. สร้าง client
client, err := NewK8sClient()
if err != nil {
    log.Fatal(err)
}

// 2. List deployments
deployments, err := client.ListDeployments(ctx, "default")
for _, d := range deployments {
    fmt.Printf("Deployment: %s (replicas: %d)\n", d.Name, *d.Spec.Replicas)
}

// 3. Scale deployment  
err = client.ScaleDeployment(ctx, "default", "my-app", 5)

// 4. Watch pods
go client.WatchPods(ctx, "default")

// 5. Use informer for caching
client.UseInformer(ctx)
`)
    
    _ = ctx
}
```

---

## 6. Custom Controller with Work Queue

```go
// controller/workqueue_controller.go
package main

import (
    "context"
    "fmt"
    "time"
    
    "k8s.io/client-go/util/workqueue"
)

// WorkQueueController implements controller with work queue
type WorkQueueController struct {
    queue      workqueue.RateLimitingInterface
    reconciler func(key string) error
    workers    int
}

// NewWorkQueueController สร้าง controller ใหม่
func NewWorkQueueController(reconciler func(key string) error, workers int) *WorkQueueController {
    return &WorkQueueController{
        queue: workqueue.NewRateLimitingQueue(
            workqueue.NewItemExponentialFailureRateLimiter(
                5*time.Millisecond,
                1000*time.Second,
            ),
        ),
        reconciler: reconciler,
        workers:    workers,
    }
}

// Add เพิ่ม item เข้า queue
func (c *WorkQueueController) Add(key string) {
    c.queue.Add(key)
}

// Run เริ่ม controller
func (c *WorkQueueController) Run(ctx context.Context) {
    defer c.queue.ShutDown()
    
    fmt.Printf("Starting controller with %d workers\n", c.workers)
    
    // เริ่ม workers
    for i := 0; i < c.workers; i++ {
        go c.runWorker(ctx)
    }
    
    <-ctx.Done()
    fmt.Println("Controller stopping...")
}

func (c *WorkQueueController) runWorker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            if !c.processNextItem() {
                return
            }
        }
    }
}

func (c *WorkQueueController) processNextItem() bool {
    item, quit := c.queue.Get()
    if quit {
        return false
    }
    defer c.queue.Done(item)
    
    key := item.(string)
    
    if err := c.reconciler(key); err != nil {
        if c.queue.NumRequeues(item) < 5 {
            fmt.Printf("Error reconciling %s: %v, requeuing\n", key, err)
            c.queue.AddRateLimited(item)
            return true
        }
        
        fmt.Printf("Error reconciling %s: %v, dropping from queue\n", key, err)
        c.queue.Forget(item)
        return true
    }
    
    c.queue.Forget(item)
    return true
}

func main() {
    reconciledCount := 0
    
    controller := NewWorkQueueController(func(key string) error {
        reconciledCount++
        fmt.Printf("Reconciling: %s (count: %d)\n", key, reconciledCount)
        time.Sleep(100 * time.Millisecond)
        return nil
    }, 3)
    
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    // เพิ่ม items เข้า queue
    go func() {
        for i := 0; i < 20; i++ {
            key := fmt.Sprintf("default/webapp-%d", i)
            controller.Add(key)
            time.Sleep(50 * time.Millisecond)
        }
    }()
    
    controller.Run(ctx)
    fmt.Printf("Total reconciled: %d\n", reconciledCount)
}
```

---

## 7. Workshop: Complete Operator

```go
// workshop/simple_operator.go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

// MockKubernetesClient จำลอง Kubernetes client
type MockKubernetesClient struct {
    mu          sync.RWMutex
    deployments map[string]*MockDeployment
    services    map[string]*MockService
}

// MockDeployment จำลอง Kubernetes Deployment
type MockDeployment struct {
    Name      string
    Namespace string
    Replicas  int32
    Image     string
    Ready     int32
}

// MockService จำลอง Kubernetes Service
type MockService struct {
    Name      string
    Namespace string
    Port      int32
}

// NewMockClient สร้าง mock client
func NewMockClient() *MockKubernetesClient {
    return &MockKubernetesClient{
        deployments: make(map[string]*MockDeployment),
        services:    make(map[string]*MockService),
    }
}

// CreateDeployment สร้าง deployment
func (c *MockKubernetesClient) CreateDeployment(d *MockDeployment) {
    c.mu.Lock()
    defer c.mu.Unlock()
    key := fmt.Sprintf("%s/%s", d.Namespace, d.Name)
    c.deployments[key] = d
    fmt.Printf("[K8s] Created Deployment: %s\n", key)
}

// UpdateDeployment อัปเดต deployment
func (c *MockKubernetesClient) UpdateDeployment(d *MockDeployment) bool {
    c.mu.Lock()
    defer c.mu.Unlock()
    key := fmt.Sprintf("%s/%s", d.Namespace, d.Name)
    if _, ok := c.deployments[key]; ok {
        c.deployments[key] = d
        fmt.Printf("[K8s] Updated Deployment: %s\n", key)
        return true
    }
    return false
}

// GetDeployment คืน deployment
func (c *MockKubernetesClient) GetDeployment(namespace, name string) (*MockDeployment, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    key := fmt.Sprintf("%s/%s", namespace, name)
    d, ok := c.deployments[key]
    return d, ok
}

// CreateService สร้าง service
func (c *MockKubernetesClient) CreateService(s *MockService) {
    c.mu.Lock()
    defer c.mu.Unlock()
    key := fmt.Sprintf("%s/%s", s.Namespace, s.Name)
    c.services[key] = s
    fmt.Printf("[K8s] Created Service: %s\n", key)
}

// SimpleOperator operator อย่างง่าย
type SimpleOperator struct {
    client      *MockKubernetesClient
    resources   map[string]*WebAppResource
    mu          sync.RWMutex
}

// WebAppResource แสดง custom resource
type WebAppResource struct {
    Name      string
    Namespace string
    Replicas  int32
    Image     string
    Port      int32
    Status    string
}

// NewSimpleOperator สร้าง operator ใหม่
func NewSimpleOperator(client *MockKubernetesClient) *SimpleOperator {
    return &SimpleOperator{
        client:    client,
        resources: make(map[string]*WebAppResource),
    }
}

// Apply สร้างหรืออัปเดต WebApp
func (op *SimpleOperator) Apply(resource *WebAppResource) {
    op.mu.Lock()
    key := fmt.Sprintf("%s/%s", resource.Namespace, resource.Name)
    op.resources[key] = resource
    op.mu.Unlock()
    
    op.reconcile(resource)
}

// reconcile จัดการ desired vs actual state
func (op *SimpleOperator) reconcile(resource *WebAppResource) {
    fmt.Printf("\n[Operator] Reconciling: %s/%s\n", resource.Namespace, resource.Name)
    
    // ตรวจสอบ Deployment
    existing, exists := op.client.GetDeployment(resource.Namespace, resource.Name)
    
    if !exists {
        // สร้าง Deployment
        op.client.CreateDeployment(&MockDeployment{
            Name:      resource.Name,
            Namespace: resource.Namespace,
            Replicas:  resource.Replicas,
            Image:     resource.Image,
            Ready:     0,
        })
        
        // สร้าง Service
        op.client.CreateService(&MockService{
            Name:      resource.Name,
            Namespace: resource.Namespace,
            Port:      resource.Port,
        })
        
        resource.Status = "Creating"
    } else {
        // อัปเดตถ้า spec เปลี่ยน
        if existing.Replicas != resource.Replicas || existing.Image != resource.Image {
            op.client.UpdateDeployment(&MockDeployment{
                Name:      resource.Name,
                Namespace: resource.Namespace,
                Replicas:  resource.Replicas,
                Image:     resource.Image,
                Ready:     existing.Ready,
            })
            resource.Status = "Updating"
        } else {
            resource.Status = "Ready"
        }
    }
    
    fmt.Printf("[Operator] Status: %s\n", resource.Status)
}

// Watch จำลองการ watch events
func (op *SimpleOperator) Watch(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            op.mu.RLock()
            for _, resource := range op.resources {
                op.reconcile(resource)
            }
            op.mu.RUnlock()
        }
    }
}

func main() {
    client := NewMockClient()
    operator := NewSimpleOperator(client)
    
    ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
    defer cancel()
    
    // เริ่ม watch goroutine
    go operator.Watch(ctx)
    
    // สร้าง WebApp resources
    fmt.Println("=== Simple Kubernetes Operator Demo ===")
    
    operator.Apply(&WebAppResource{
        Name:      "my-web-app",
        Namespace: "default",
        Replicas:  3,
        Image:     "nginx:1.21",
        Port:      8080,
    })
    
    time.Sleep(2 * time.Second)
    
    // อัปเดต replicas
    fmt.Println("\n--- Scaling up replicas ---")
    operator.Apply(&WebAppResource{
        Name:      "my-web-app",
        Namespace: "default",
        Replicas:  5,
        Image:     "nginx:1.21",
        Port:      8080,
    })
    
    time.Sleep(2 * time.Second)
    
    // อัปเดต image
    fmt.Println("\n--- Rolling update ---")
    operator.Apply(&WebAppResource{
        Name:      "my-web-app",
        Namespace: "default",
        Replicas:  5,
        Image:     "nginx:1.22",
        Port:      8080,
    })
    
    // สร้าง WebApp ที่สอง
    operator.Apply(&WebAppResource{
        Name:      "api-service",
        Namespace: "production",
        Replicas:  2,
        Image:     "myapi:v1.0",
        Port:      3000,
    })
    
    time.Sleep(3 * time.Second)
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **CRDs** - การสร้าง Custom Resource Definitions เพื่อขยาย Kubernetes API
2. **Operator Pattern** - หลักการทำงานของ Kubernetes Operators
3. **controller-runtime** - Framework สำหรับสร้าง operators
4. **Admission Webhooks** - Validating และ Mutating webhooks
5. **client-go** - Kubernetes Go client library
6. **Work Queue Controller** - การใช้ work queue ใน controllers

### Key Takeaways

- **Operators = Controller + CRD** - ขยาย Kubernetes ด้วย domain knowledge
- **Reconcile Loop** - Loop ที่ทำให้ actual state = desired state
- **Owner References** - ใช้เพื่อ cascade deletion
- **Informers** - ใช้ caching เพื่อ performance ที่ดีขึ้น
- **Leader Election** - ใช้เมื่อรัน operator หลาย instances
