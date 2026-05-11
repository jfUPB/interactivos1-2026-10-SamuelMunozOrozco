# Unidad 8

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

### Que se tenia que hacer

* Teniamos que integrar todo lo aprendido desde la unidad 5 hasta esta ultima unidad, debiamos ser capaces de integrar tres fuentes de informacion diferentes, que manden datos al mismo tiempo y que estos puedan ser leidos correctamente y mandados todos juntos por el bridgeserver al frontend

### Arquitectura general del sistema

#### Microbit

```
MICROBIT
↓
MicrobitAsciiAdapter
↓
                 ┌───────────────────────┐
STRUDEL ───────→ │                       │
↓                │     bridgeServer      │
StrudelAdapter   │                       │
↓                └───────────────────────┘
                                   ↓
OPEN STAGE CONTROL                WebSocket
↓                                  ↓
OpenStageControlAdapter     bridgeClient.js
                                   ↓
                                FSMTask
                                   ↓
                             updateLogic()
                                   ↓
                             drawRunning()
                                   ↓
                         VISUALES EN p5.js
```

### Rol de cada fuente

| Fuente | Qué controla | Cómo entra al sistema | Por qué se usa |
|---|---|---|---|
| micro:bit | Cambio de formas visuales y partículas | Serial → MicrobitAsciiAdapter | Permite interacción física y gestual |
| Strudel | Generación de eventos musicales y temporales | WebSocket → StrudelAdapter | Controla sincronización audiovisual |
| Open Stage Control | Parámetros globales visuales | OSC → OpenStageControlAdapter | Permite manipulación performática en tiempo real |






## Bitácora de reflexión
