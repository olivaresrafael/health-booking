---
description: Implementa una tarea del roadmap por su ID (ej. T-005)
---

Implementa la tarea $ARGUMENTS de `docs/ROADMAP.md`.

Pasos:
1. Lee la tarea, sus dependencias y los documentos que menciona. Si una dependencia no está hecha, dilo y detente.
2. Propón un plan corto (archivos a crear o cambiar) y espera mi confirmación si el cambio toca el esquema de base de datos o la seguridad.
3. Implementa en pasos pequeños, con pruebas para las reglas del motor.
4. Ejecuta `pnpm typecheck && pnpm lint && pnpm test` y corrige lo que falle.
5. Marca la tarea como hecha en `docs/ROADMAP.md` y resume qué cambió y qué queda pendiente.
