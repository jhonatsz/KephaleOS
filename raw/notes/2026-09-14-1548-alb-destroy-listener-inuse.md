---
type: source
status: compiled
source-kind: note
created: 2026-09-14
captured-at: 2026-09-14T15:48:33
compiled-at: 2026-10-01
compiled-into:
  - "[[wiki/technologies/aws-elbv2-alb-nlb]]"
---

# Question

find why this cant destroy

# Terraform output

aws_lb_listener.lb_listener: Destroying... [id=arn:aws:elasticloadbalancing:us-west-2:872194582181:listener/app/alb-prod-unstractlte/c3dfd8eddd4db22b/d7fede39305af1dd]
aws_lb_listener.lb_listener_unstractlte_http: Destroying... [id=arn:aws:elasticloadbalancing:us-west-2:872194582181:listener/app/alb-prod-unstractlte/c3dfd8eddd4db22b/3af52c3eacbb3ae4]
╷
│ Error: deleting Listener (arn:aws:elasticloadbalancing:us-west-2:872194582181:listener/app/alb-prod-unstractlte/c3dfd8eddd4db22b/3af52c3eacbb3ae4): ResourceInUse: Listener port '80' is in use by registered target 'arn:aws:elasticloadbalancing:us-west-2:872194582181:loadbalancer/app/alb-prod-unstractlte/c3dfd8eddd4db22b' and cannot be removed.
│     status code: 400, request id: 5a2c91dc-adb9-4cd5-b1a0-211d0618e99d
│
│
╵
╷
│ Error: deleting Listener (arn:aws:elasticloadbalancing:us-west-2:872194582181:listener/app/alb-prod-unstractlte/c3dfd8eddd4db22b/d7fede39305af1dd): ResourceInUse: Listener port '443' is in use by registered target 'arn:aws:elasticloadbalancing:us-west-2:872194582181:loadbalancer/app/alb-prod-unstractlte/c3dfd8eddd4db22b' and cannot be removed.
│     status code: 400, request id: 3fe6c1fe-7631-49b8-975a-d6070cb51591
│
│
╵
