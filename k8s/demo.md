Step 1 — Apply the crash pod and alert rule:                                                                                     
  kubectl apply -f k8s/dev/crash-demo.yaml                                                                                         
  kubectl apply -f k8s/monitoring/crash-demo-alert.yaml                                                                            
                                                                                                                                   
  Step 2 — Sync ArgoCD to pick up the new alertmanager webhook config:                                                             
  argocd app sync monitoring --insecure --server localhost:8081                                                                    
                                                                                                                                   
  Step 3 — Watch the pod crash:                                                                                                    
  kubectl get pod crash-demo -n dev -w                                                                                             
                                                                                                                                   
  Step 4 — Watch for the alert in Prometheus (~2 min):                                                                             
  kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090                                           
  # open http://localhost:9090/alerts → look for CrashDemoPodCrashing                                                              
                                                                                                                                   
  Step 5 — Check webhook.site — the alert payload will appear there within ~1-2 minutes of the pod crashing. 