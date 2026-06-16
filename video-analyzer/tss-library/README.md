# TSS Model Library

Coloca aquí los videos modelo (.mp4) por categoría. La app los lee desde
`/public/tss-library/<categoria>/`.

Categorías (carpetas):
- take-off/
- bottom-turn/
- top-turn/
- cutback/
- floater/
- barrel-line/
- air/

## Cómo agregar un video

1. Copia el archivo, por ejemplo: `take-off/model-1.mp4`
2. Asegúrate de que exista una entrada en `lib/library.ts` apuntando a ese `src`.
3. Recomendado: H.264 (.mp4) para máxima compatibilidad con iPad Safari.

Los nombres del catálogo actual esperan archivos `model-1.mp4` (y `model-2.mp4`
en Take Off). Si un archivo no existe, la app muestra un aviso en el panel
en lugar de romperse.
