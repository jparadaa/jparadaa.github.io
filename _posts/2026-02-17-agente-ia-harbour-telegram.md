# Construyendo un Asistente de IA Empresarial con Harbour y Telegram

## El Problema

En nuestra empresa, el equipo de ventas y gerencia necesita consultar información constantemente: ventas del día, inventario disponible, reportes de clientes... pero siempre requiere estar frente a la computadora, abrir el sistema, navegar por módulos, o peor aún, llamar a alguien para que busque el dato.

La pregunta era: **¿Cómo dar acceso rápido a información crítica sin forzar un cambio tecnológico radical?**

---

## La Solución: Agente de IA Empresarial vía Telegram

Decidí crear un agente de IA que funciona a través de Telegram, permitiendo consultas en lenguaje natural desde cualquier lugar. La arquitectura combina:

- **Harbour** - Nuestro stack existente
- **Telegram** - Interfaz familiar para usuarios
- **Ollama (llama3.2:3b)** - IA local para procesamiento de lenguaje natural
- **Base de datos** - Sistema de producción existente

---

## Arquitectura Técnica

**Stack completo:**
```
Usuario escribe en Telegram: "¿Cuánto vendimos hoy?"
↓
Agente en Harbour (Polling):
- Autenticación con lista blanca
- Control de roles
- Memoria de conversación
↓
Ollama (IA Local):
- Detección de intención
- Procesamiento contextual
- Respuestas en lenguaje natural
↓
Base de datos:
- Datos de producción
- Stored procedures
- Logging de auditoría
```

---

## Funcionalidades Implementadas

### 1. Seguridad Empresarial

- Lista blanca de usuarios autorizados
- Sistema de roles (admin, user, readonly)
- Registro automático de intentos de acceso
- Auditoría completa de actividad

### 2. Detección Inteligente de Intención

El agente clasifica automáticamente las consultas:

- **Ventas:** "¿Cuánto vendimos en enero?"
- **Inventario:** "¿Cuánto stock hay de filtros?"
- **Productos:** "Lista de productos disponibles"
- **Chat general:** Conversación natural

### 3. Memoria Contextual

Mantiene historial de 20 mensajes por usuario (fácilmente ajustable mediante configuración, pero creemos que es suficiente para una interacción natural con el agente):
```
Usuario: "Me llamo Javier, dame las ventas"
Agente: "Claro Javier, las ventas son..."
[Más tarde]
Usuario: "¿Recuerdas mi nombre?"
Agente: "Sí, eres Javier"
```

### 4. Monitoreo y Métricas

- Tiempo de respuesta medido
- Log de todas las consultas
- Tracking de usuarios más activos

---

## Decisiones Técnicas Clave

### ¿Por qué Telegram y no una web app?

**Movilidad:** El equipo está constantemente fuera de oficina. Telegram está siempre en sus celulares, no requiere VPN ni abrir navegadores.

**Adopción:** Ya usan Telegram diariamente para comunicación interna. Cero curva de aprendizaje.

**Simplicidad:** Escriben como si fuera un mensaje normal. No hay interfaces complicadas.

### ¿Por qué IA local (Ollama) en lugar de APIs comerciales?

**Privacidad:** Datos empresariales sensibles no salen del servidor.

**Costo:** $0 por request vs. facturación por token de OpenAI/Claude.

**Control:** Ajustes y optimizaciones sin límites de API.

**Modelo elegido:** Estamos utilizando llama3.2:3b, un modelo optimizado para los recursos físicos con los que contamos en esta fase inicial. En nuestras pruebas ha demostrado ser suficiente para entender las consultas y generar respuestas coherentes. Es evidente que con mayores recursos físicos (GPU dedicada o servidores especializados) podríamos utilizar modelos más potentes con mayor capacidad de razonamiento y velocidad de respuesta.

### ¿Por qué Polling en lugar de Webhooks?

**Simplicidad:** No requiere servidor HTTP expuesto, SSL, ni configuración de dominios.

**Debugging:** Logs visibles en tiempo real, fácil de detener/reiniciar.

**Suficiente para nuestro entorno:** Con 10-20 usuarios máximo en nuestro contexto empresarial, polling cumple perfectamente los requisitos sin agregar complejidad innecesaria.

---

## Código Base: Clase de Telegram para Harbour

Todo esto fue posible gracias a la clase de Telegram para Harbour desarrollada por [Riztán Gutiérrez](https://github.com/riztan/hbtelegram), que proporciona una interfaz limpia y robusta para interactuar con la API de Telegram desde Harbour.

Su implementación maneja:

- Polling con long-timeout
- Envío de mensajes con formato HTML/Markdown
- Manejo de errores y timeouts
- Soporte para adjuntos y multimedia

**Sin esta base sólida, el proyecto habría tomado semanas más de desarrollo.**

---

## Desafíos y Aprendizajes

### 1. Velocidad de respuesta

Las pruebas actuales se están realizando en equipos sin GPU dedicada. El tiempo de respuesta es completamente aceptable para esta fase de desarrollo y pruebas. Sin embargo, somos conscientes de que para un ambiente de producción será necesario invertir en recursos físicos más robustos (GPU dedicada) o considerar servicios de VPS especializados en proyectos de IA que ofrecen infraestructura optimizada para este tipo de cargas de trabajo.

### 2. Detección de intención

Uno de los componentes más críticos del agente es su capacidad para entender qué está pidiendo realmente el usuario. El modelo llama3.2:3b que utilizamos ha demostrado buena efectividad en clasificar las consultas entre categorías (ventas, inventario, productos, o conversación general). La elección de este modelo específico responde a los recursos físicos disponibles para estas pruebas iniciales. En nuestras evaluaciones ha resultado suficiente, aunque es claro que modelos más grandes y potentes ofrecerían mayor precisión y capacidades avanzadas, especialmente si se cuenta con hardware más robusto que permita tiempos de respuesta más rápidos.

### 3. Contexto en respuestas estructuradas

Mantener el contexto de la conversación mientras el agente consulta datos específicos requirió un diseño cuidadoso. La solución implementada asegura que las respuestas sean naturales y consideren toda la información previa de la conversación, incluyendo el nombre del usuario y preferencias expresadas anteriormente.

---

## Resultados Hasta Ahora

**Estado actual (Desarrollo Inicial):**

- ✅ Agente funcional con seguridad empresarial
- ✅ Detección de intención operativa
- ✅ Memoria contextual configurable
- ✅ Sistema de autenticación y roles
- ⏳ Pendiente: Conexión a datos reales de producción

**Próximos pasos:**

1. Conectar consultas reales a base de datos de producción
2. Probar con 3-5 usuarios beta (2-4 semanas)
3. Recopilar métricas de uso y feedback
4. Evaluar inversión en infraestructura especializada
5. Expandir funcionalidades según necesidades reales

---

## Conclusiones

Este proyecto demuestra que es posible modernizar procesos sin reescribir sistemas completamente. La combinación de Harbour (nuestro stack existente), Telegram (interfaz familiar), y Ollama (IA local) crea una solución práctica y de bajo costo.

**Lo más importante:** Validar el valor real antes de invertir en infraestructura costosa. Un agente que funciona y es útil, aunque no sea instantáneo, es mejor que una solución perfecta que nadie usa.

---

## Agradecimientos

Especial agradecimiento a **Riztán Gutiérrez** por desarrollar y liberar como código abierto su [clase de Telegram para Harbour](https://github.com/riztan/hbtelegram), componente fundamental que proporciona una interfaz robusta y bien diseñada para la comunicación con la API de Telegram. Sin esta base técnica, el desarrollo del proyecto habría requerido considerablemente más tiempo.

Agradecimiento también a **Claude (Anthropic)** por la guía técnica durante el desarrollo.

---

**¿Tienes experiencia con agentes de IA en Harbour? ¿Has implementado soluciones similares? Me encantaría leer tus comentarios.**
