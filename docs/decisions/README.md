# Decision Records

Architecture decision records for decisions that bind more than one repo.
Each one's settled rule lives under `base/`; the record keeps the
reasoning. Format and lifecycle:
[`base/shared/decision-records.md`](../../base/shared/decision-records.md).

| # | Decision | Status |
|---|---|---|
| [0001](0001-select-app-build-mode-with-make-mode.md) | Select the app's build/run mode with `MODE=dev\|prod` on the app targets | Accepted |
| [0002](0002-group-the-domain-model-in-one-domain-folder-split-by-feature.md) | Group the domain model in one `domain/` folder, split by feature | Superseded by [0003](0003-package-backend-code-by-component.md) |
| [0003](0003-package-backend-code-by-component.md) | Package backend code by component | Superseded by [0004](0004-package-backend-code-by-component-with-tests-split-by-tier.md) |
| [0004](0004-package-backend-code-by-component-with-tests-split-by-tier.md) | Package backend code by component, with tests split by tier | Accepted |
| [0005](0005-reach-persistence-through-domain-defined-repository-interfaces.md) | Reach persistence through repository interfaces defined with the domain model | Superseded by [0006](0006-give-every-repository-the-same-crud-shape.md) |
| [0006](0006-give-every-repository-the-same-crud-shape.md) | Give every repository the same CRUD shape | Accepted |
