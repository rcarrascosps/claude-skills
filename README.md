# Claude Code Skills - API Development Automation

Este repositorio contiene una colección de skills personalizadas para Claude Code, diseñadas específicamente para automatizar y optimizar tareas relacionadas con el desarrollo de APIs.

## 📋 Tabla de Contenidos

- [¿Qué son las Skills de Claude Code?](#qué-son-las-skills-de-claude-code)
- [Instalación](#instalación)
- [Skills Disponibles](#skills-disponibles)
- [Cómo Usar las Skills](#cómo-usar-las-skills)
- [Crear tus Propias Skills](#crear-tus-propias-skills)
- [Contribuir](#contribuir)

## ¿Qué son las Skills de Claude Code?

Las skills son conjuntos de instrucciones y conocimientos especializados que extienden las capacidades de Claude Code para realizar tareas específicas de manera más eficiente y consistente. En este caso, nuestras skills están enfocadas en automatizar procesos comunes en el desarrollo de APIs.

## Instalación

### Método 1: Instalación Manual

1. **Localiza tu directorio de skills de Claude Code:**
   - **macOS/Linux**: `~/.claude/skills/`
   - **Windows**: `%USERPROFILE%\.claude\skills\`

2. **Clona este repositorio:**
```bash
   cd ~/.claude/skills/  # o la ruta correspondiente en Windows
   git clone https://github.com/tu-usuario/claude-code-api-skills.git
```

3. **Copia las skills individuales:**
```bash
   cp -r claude-code-api-skills/skills/* .
```

### Método 2: Instalación por Skill Individual

Si solo quieres instalar skills específicas:

1. **Navega al directorio de skills:**
```bash
   cd ~/.claude/skills/
```

2. **Copia solo las carpetas de las skills que necesites:**
```bash
   cp -r /ruta/al/repo/skills/nombre-de-skill ./
```

### Verificar la Instalación

Después de instalar las skills:

1. Reinicia Claude Code si está en ejecución
2. Las skills deberían cargarse automáticamente
3. Puedes verificar preguntándole a Claude: "¿Qué skills tienes disponibles para desarrollo de APIs?"

## Skills Disponibles

| Skill | Descripción | Casos de Uso |
|-------|-------------|--------------|
| `mulesoft-documentor` | Genera la documentación de una aplicacion MuleSoft | Crear documentación de aplicación MuleSoft cuando esta no existe |
| `mulesoft-api-patterns` | Genera proyectos de mulesoft basado en patrones de diseño de APIs | Genera códigos de ejemplo basado en principios de Diseño |

*(Esta tabla se actualizará conforme se agreguen más skills)*

## Cómo Usar las Skills

Las skills se activan automáticamente cuando Claude Code detecta que tu solicitud está relacionada con su dominio. Sin embargo, también puedes invocarlas explícitamente:

### Ejemplos de Uso

**Generar una especificación OpenAPI:**
```
Usando la skill de api-spec-generator, crea una especificación OpenAPI 3.0 
para un API de gestión de usuarios con endpoints CRUD
```

**Crear un nuevo endpoint:**
```
Crea un endpoint POST /api/products que acepte nombre, precio y descripción, 
validando que el precio sea positivo
```

**Generar tests:**
```
Genera tests unitarios para el endpoint GET /api/users/:id usando Jest
```

## Crear tus Propias Skills

### Estructura de una Skill

Cada skill debe seguir esta estructura:
```
nombre-de-tu-skill/
├── SKILL.md          # Archivo principal (obligatorio)
├── templates/        # Templates opcionales
├── examples/         # Ejemplos de uso
└── resources/        # Recursos adicionales
```

### Formato del SKILL.md

Tu archivo `SKILL.md` debe incluir:
```markdown
# Nombre de la Skill

## Descripción
[Breve descripción de qué hace la skill]

## Cuándo Usar Esta Skill
- [Escenario 1]
- [Escenario 2]
- [Escenario 3]

## Capacidades
- [Capacidad 1]
- [Capacidad 2]

## Instrucciones para Claude
[Instrucciones detalladas paso a paso de cómo ejecutar la skill]

## Ejemplos de Uso

### Ejemplo 1: [Título]
**Input del usuario:**
```
[Ejemplo de lo que el usuario pediría]
```

**Acción esperada:**
[Lo que Claude debería hacer]

## Consideraciones Especiales
- [Nota importante 1]
- [Nota importante 2]

## Dependencias
- [Si requiere herramientas específicas]
- [Si depende de otras skills]
```

### Mejores Prácticas

1. **Sé específico**: Proporciona instrucciones claras y detalladas
2. **Incluye ejemplos**: Los ejemplos ayudan a Claude a entender el contexto
3. **Define el alcance**: Especifica claramente cuándo usar (y cuándo no usar) la skill
4. **Mantén la modularidad**: Cada skill debe tener un propósito bien definido
5. **Documenta las dependencias**: Indica si requiere librerías, frameworks o herramientas específicas

### Ejemplo Completo

Revisa las skills existentes en este repositorio como referencia. Por ejemplo, `api-spec-generator/SKILL.md` es un buen punto de partida.

## Contribuir

¡Las contribuciones son bienvenidas! Si tienes una skill útil para el desarrollo de APIs:

1. **Fork** este repositorio
2. **Crea** una nueva rama (`git checkout -b feature/nueva-skill`)
3. **Agrega** tu skill siguiendo la estructura descrita
4. **Commit** tus cambios (`git commit -m 'Add: skill para [funcionalidad]'`)
5. **Push** a la rama (`git push origin feature/nueva-skill`)
6. **Abre** un Pull Request

### Lineamientos para Contribuciones

- Asegúrate de que tu skill esté bien documentada
- Incluye ejemplos de uso claros
- Verifica que no duplique funcionalidad existente
- Sigue las convenciones de nomenclatura (kebab-case para nombres de carpetas)
- Actualiza la tabla de "Skills Disponibles" en este README

## Estructura del Repositorio
```
.
├── README.md
├── skills/
│   ├── api-spec-generator/
│   │   ├── SKILL.md
│   │   └── templates/
│   ├── endpoint-creator/
│   │   ├── SKILL.md
│   │   └── examples/
│   └── ... (más skills)
└── docs/
    ├── creating-skills.md
    └── best-practices.md
```

## Soporte

Si encuentras problemas o tienes preguntas:

- 📝 Abre un [Issue](https://github.com/tu-usuario/claude-code-api-skills/issues)
- 💬 Consulta la [documentación oficial de Claude Code](https://docs.claude.com)
- 🤝 Únete a las discusiones en la sección de Discussions

## Licencia

TBD

## Reconocimientos

- Desarrollado para la comunidad de Claude Code
- Inspirado por las mejores prácticas de desarrollo de APIs

---

**Nota**: Este proyecto no está oficialmente afiliado con Anthropic. Es un proyecto comunitario para mejorar la experiencia de desarrollo con Claude Code.
