# Prompt 1
Dado mi enunciado en @assignment.md, necesito crear un pipeline en GitHub Actions para automatizar los pasos indicados.

Necesito que me recuerdes cómo tenía que configurar el workflow de GitHub Actions en un archivo `.github/workflows/ci.yml` y documentar los prompts utilizados para generar cada paso del pipeline.

Resultado: se revisó el repositorio (assignment.md, .github/workflows/ci.yml vacío, package.json de backend/frontend) para confirmar que no existía un pipeline previo, y se generó el workflow desde cero con el trigger `pull_request` (types: opened, synchronize, reopened) para que se dispare en cada push a una rama con PR abierto, más los tres jobs encadenados descritos en los siguientes prompts. 

# Prompt 2 — Trigger del pipeline
El pipeline debe dispararse ante un "push a una rama con un Pull Request abierto". En GitHub Actions esto no es un evento `push` con condición, sino el evento `pull_request` con `types: [opened, synchronize, reopened]`: `synchronize` es el que se dispara en cada push posterior sobre la rama de un PR ya abierto, `opened`/`reopened` cubren el primer push. Se usó este trigger a nivel de workflow para que aplique a los tres jobs.

# Prompt 3 — Tests de backend
Genera el job `test` del pipeline: debe hacer checkout del código, configurar Node.js 20 con caché de npm apuntando a `backend/package-lock.json`, instalar dependencias con `npm ci` dentro de `backend/` (usando `working-directory`/`defaults.run`) y ejecutar `npm test` (que en `backend/package.json` corre Jest). Este job es el primero de la cadena y los siguientes (`build`, `deploy`) dependen de él vía `needs`.

# Prompt 4 — Build del backend
Genera el job `build`, dependiente de `test` (`needs: test`), que repite checkout + setup de Node 20 + `npm ci` en `backend/`, ejecuta `npm run build` (compila TypeScript con `tsc` a `backend/dist` según el script del package.json) y sube el resultado como artefacto (`actions/upload-artifact`) incluyendo `dist/`, `package.json` y `package-lock.json`, para que el job de deploy no tenga que volver a compilar.

# Prompt 5 — Despliegue del backend en EC2
Genera el job `deploy`, dependiente de `build` (`needs: build`), que descarga el artefacto generado y lo copia a la instancia EC2 por SSH/SCP (`appleboy/scp-action`) usando secretos del repositorio (`EC2_HOST`, `EC2_USERNAME`, `EC2_SSH_KEY`, `EC2_TARGET_DIR`), y luego se conecta por SSH (`appleboy/ssh-action`) para instalar dependencias de producción (`npm ci --omit=dev`) y reiniciar el proceso con `pm2 restart backend` (o arrancarlo si no existe). Los secretos deben configurarse en GitHub (Settings → Secrets and variables → Actions) antes de que el job pueda ejecutarse con éxito.