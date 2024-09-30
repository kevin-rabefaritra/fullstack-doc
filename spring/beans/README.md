# Spring Beans

## Key points
- Objects that are instanciated and managed by the Spring IoC (Inversion of Control) container.
- IoC is a process where an object defines its dependencies without creating them.

## Lifecycle

(1) Spring container is started

(2) `setBeanName`, `setBeanFactory`, `setApplicationContext`

(3) `BeanPostProcessors`

(4) `InitializingBean.afterPropertiesSet` then **custom init method** `@PostContruct`.

(5) Spring container is shut down

(6) `DisposableBean.destroy` then **custom destroy method** `@PreDestroy`.