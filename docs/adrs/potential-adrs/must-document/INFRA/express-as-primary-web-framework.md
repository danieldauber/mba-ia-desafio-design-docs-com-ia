# Potential ADR: Express 4.x as Primary Web Framework

**Module**: INFRA
**Category**: Technology
**Priority**: Must Document (Score: 140/150)
**Date Identified**: 2026-10-06

---

## What Was Identified

Express 4.x is the sole HTTP framework bootstrapping the application. The pattern was introduced with the initial repository commit on 2026-06-24 — the entire middleware pipeline, routing, and application lifecycle are built on Express primitives.

The `buildApp(deps)` factory in `src/app.ts` composes the full middleware stack (JSON body parser at 1MB limit, request logger, API router, 404 catch-all, error middleware) directly onto an Express `Application` instance. All five business modules expose routes via Express `Router` objects, and the centralized error handler is an Express `ErrorRequestHandler`. There is no abstraction layer between the framework and the application code; Express types (`Request`, `Response`, `NextFunction`, `RequestHandler`, `ErrorRequestHandler`) are used directly across `src/middlewares/` and all route files.

The choice of Express over alternatives such as Fastify, NestJS, or Hono is not documented in the codebase. Given the team-size context noted in `decisions.md` (where Kafka/RabbitMQ were rejected as overengineering), Express's minimal footprint and wide ecosystem familiarity align with the stated philosophy. The framework is currently on version 4.21.1 — Express 5 was in RC at time of adoption and is not in use.

## Why This Might Deserve an ADR

- **Impact**: Every HTTP feature, middleware, route handler, and error contract in the system is built on Express APIs. Replacing or upgrading the framework touches every module.
- **Trade-offs**: Express 4.x provides maximum flexibility but no built-in validation, serialization, OpenAPI generation, or TypeScript-first request typing. Those gaps are filled by Zod, custom middleware, and manual typing — a deliberate set of choices worth capturing.
- **Complexity**: The framework version boundary matters: Express 5 changes error handling for async route handlers (no more `next(err)` requirement), which would affect all existing async controllers.
- **Team Knowledge**: Every engineer extending the API must understand Express middleware chaining, error propagation via `next(err)`, and how `buildApp` composes the pipeline.
- **Future Implications**: The planned `src/worker.ts` process will NOT use Express (it is a background poller), making it important to document that Express is the API server only, not a universal application framework.

## Evidence Found in Codebase

### Key Files
- [`src/app.ts`](../../../../src/app.ts) - Lines 1-76 — full Express application factory and DI composition root
- [`src/server.ts`](../../../../src/server.ts) - Lines 1-27 — Express server bootstrap, graceful shutdown
- [`src/middlewares/error.middleware.ts`](../../../../src/middlewares/error.middleware.ts) - Lines 14-65 — Express `ErrorRequestHandler` signature
- [`src/middlewares/validate.middleware.ts`](../../../../src/middlewares/validate.middleware.ts) - Lines 1-37 — Express `RequestHandler` for Zod validation

### Code Evidence
```typescript
// src/app.ts:55-68
export function buildApp(deps: AppDependencies): Express {
  const app = express();

  app.disable('x-powered-by');
  app.use(express.json({ limit: '1mb' }));
  app.use(requestLogger);

  app.get('/health', (_req, res) => {
    res.status(200).json({ status: 'ok' });
  });

  const controllers = buildControllers(deps.prisma);
  app.use('/api/v1', buildApiRouter(controllers));
  // ...
}
```

```typescript
// src/middlewares/error.middleware.ts:14
export const errorMiddleware: ErrorRequestHandler = (err, req, res, _next) => {
```

### Impact Analysis
- Introduced: 2026-06-24 (initial commit)
- Modified: 1 commit (stable since introduction)
- Affects: all 7 route-level files, all 5 middleware files, both entry points
- All modules depend on Express types directly

### Alternatives (if observable)
No alternatives are explicitly mentioned in code comments or commit messages. The `decisions.md` context file documents the team's preference for avoiding overengineering, which is consistent with choosing Express over an opinionated framework such as NestJS.

## Questions to Address in ADR (if created)

- Why Express 4.x over Fastify (performance), NestJS (structure), or Hono (edge-ready TypeScript)?
- Was Express 5 evaluated? Why stay on v4?
- Is the absence of a framework adapter layer intentional (direct Express dependency)?
- How will the worker process architecture differ from the Express server, and should they share any infrastructure primitives?

## Related Potential ADRs
- [`mysql-8-via-docker-compose-as-primary-database.md`](./mysql-8-via-docker-compose-as-primary-database.md)

## Additional Notes
The `x-powered-by` header is explicitly disabled (`app.disable('x-powered-by')`), indicating security awareness. The JSON body limit of 1MB is hardcoded — not env-configurable — which may warrant a note in the ADR about the rationale.
