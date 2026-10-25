# ApnaArea Roadmap

Mark a step `[x]` when its branch is merged. `/create-spec` reads this file.
Steps are small units; expect several per day.

## Foundation
- [ ] 01 Repo scaffold, Docker Compose (Postgres, Redis), schemas and design doc
- [ ] 02 Location data ETL (states, constituencies, MLAs, MPs) into `problems` schema

## problem-service
- [ ] 03 Create and read problem with mandatory location fields
- [ ] 04 Feed with keyset pagination, filters and constituency search
- [ ] 05 MP/MLA lookup by constituency

## auth-service
- [ ] 06 Google login and JWT issuing
- [ ] 07 JWT validation in problem-service; protect problem creation

## vote-service
- [ ] 08 Upvote and remove upvote with duplicate protection
- [ ] 09 Redis Stream event `vote.cast` and `upvote_count` consumer
- [ ] 10 Feed cache and rate limiting in Redis

## web
- [ ] 11 Frontend scaffold and Google sign-in
- [ ] 12 Feed with infinite scroll and filters
- [ ] 13 Create-problem form and MP/MLA card

## Deploy and finish
- [ ] 14 Dockerfiles and Kubernetes manifests
- [ ] 15 Deploy to k3s with HTTPS
- [ ] 16 CI with path filters and headless review
- [ ] 17 README, ARB-style design doc, `scaling.md`, failure log review