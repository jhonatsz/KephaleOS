---
type: source
status: compiled
source-kind: note
created: 2026-09-14
captured-at: 2026-09-14T15:18:31
compiled-at: 2026-10-01
compiled-into:
  - "[[wiki/technologies/aws-elbv2-alb-nlb]]"
---

this settings always have change update on all alb

  # aws_lb_listener.lb_listener will be updated in-place
  ~ resource "aws_lb_listener" "lb_listener" {
        id                = "arn:aws:elasticloadbalancing:us-west-2:872194582181:listener/app/alb-uat-taskmanager/4f071c53574836c2/3039ea7e83a21e0b"
        tags              = {
            "Environment" = "UAT"
            "Project"     = "TASKMANAGER"
        }
        # (7 unchanged attributes hidden)

      ~ default_action {
          - target_group_arn = "arn:aws:elasticloadbalancing:us-west-2:872194582181:targetgroup/tg-uat-taskmanager-rails/c02b5f4b8e71db27" -> null
            # (2 unchanged attributes hidden)

          + forward {
              + stickiness {
                  + duration = 15
                  + enabled  = true
                }
              + target_group {
                  + arn    = "arn:aws:elasticloadbalancing:us-west-2:872194582181:targetgroup/tg-uat-taskmanager-rails/c02b5f4b8e71db27"
                  + weight = 1
                }
            }
        }
    }
