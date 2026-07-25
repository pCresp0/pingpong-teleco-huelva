# 🏓 Ping-Pong Teleco Huelva

Web para gestionar la liga interna de ping-pong de un grupo de amigos (ex-Teleco de Huelva) durante un viaje: calendario, resultados, clasificación en vivo, playoffs y estadísticas — pensada para usarse desde el móvil.

**🔗 Web en producción:** https://pcresp0.github.io/pingpong-teleco-huelva/

## Qué hace

- **Liga a una vuelta**: cada jugador se enfrenta una vez a todos los demás. Partidos a 11 puntos, sin empate posible (hay que ganar por 2 o más).
- **Clasificación automática**: se recalcula sola al guardar cada resultado (puntos de liga, victorias, derrotas, puntos a favor/en contra, diferencia y ratio).
- **Desempates**: en caso de empate a puntos, decide primero el enfrentamiento directo entre los jugadores empatados y, si también hay empate ahí, el ratio puntos a favor / puntos en contra.
- **Playoffs top 4**: al terminar la liga regular se generan las semifinales (1º vs 4º, 2º vs 3º), la final y el partido de 3º y 4º puesto.
- **Estadísticas**: puntos totales, media por partido, mejor racha, máximo anotador, ranking de regularidad (desviación de puntos) y ranking de victorias más abultadas.
- **Curiosidades**: datos random generados a partir de los resultados reales de la liga.
- **Plantilla fija**: 7 jugadores predefinidos, cada uno con un emoji que le representa.

## Jugadores

| | |
|---|---|
| 💃 Paloma | 📐 Patricia |
| 🌲 Miguel Ángel | 🤖 Escudero |
| 🔭 Pablo | 🎾 David |
| 🎵 Rocío | |

## Cómo funciona por dentro

Es una app de página única (SPA) en **React 18**, compilada de antemano con **esbuild** (sin transpilar JSX en el navegador) y estilada con **Tailwind CSS** compilado también en build time. No hay backend propio: los datos de la liga se guardan en un JSON compartido y gratuito en [jsonblob.com](https://jsonblob.com), identificado por un id que viaja en la URL (`?liga=ID`) y en el `localStorage` de cada dispositivo. Así, todo el grupo ve y edita la misma liga desde su móvil sin necesidad de cuentas ni servidor.

Se sirve como sitio estático con **GitHub Pages**, y se reconstruye sola cada vez que hay un commit nuevo en `main`.

### Estructura del repo

```
├── index.html     # HTML mínimo: carga bundle.js y styles.css
├── bundle.js      # App de React ya compilada (esbuild)
├── styles.css     # CSS de Tailwind ya compilado
└── README.md
```

## Desarrollo

El código fuente de la app (JSX) y los scripts de build no están versionados en este repo (se compilan aparte y solo se sube el resultado). Para tocar la app, edita el componente React, recompílalo con `esbuild` + `tailwindcss` CLI y sube `index.html`, `bundle.js` y `styles.css` actualizados.

---

Hecho con 🏓 por [Pablo Crespo](https://www.linkedin.com/in/pablocrespobellido).
