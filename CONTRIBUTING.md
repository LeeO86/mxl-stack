# Contributing

- Bugs and features of one media function: open the issue in that function's
  repository. This repository only links them.
- Here: corrections to the requirements, the Compose examples and the
  Kubernetes notes. Keep them in line with the function READMEs, which are the
  source of truth.
- When a function releases, update its version in `README.md` and the image
  tags in `compose/`, and add a line to `CHANGELOG.md`.
- Check before a pull request:

  ```bash
  (cd compose/single-host && docker compose --env-file example.env config -q)
  (cd compose/multi-host-rdma/host-a && docker compose --env-file example.env config -q)
  (cd compose/multi-host-rdma/host-b && docker compose --env-file example.env config -q)
  ```

- Use documentation addresses in examples (192.0.2.0/24, 198.51.100.0/24,
  example.com). No real host names, addresses or credentials.

By contributing you agree that your contribution is licensed under the MIT
License of this repository.
