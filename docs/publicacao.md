# Publicar no GitHub

Este pacote prepara o conteúdo do repositório. Nenhum repositório remoto foi criado ou atualizado automaticamente.

## Pelo Git no computador

1. Extraia o ZIP.
2. Crie um repositório vazio na sua conta do GitHub. Nome sugerido: `eve-ng-pfsense-ha-switch-redundancy`.
3. Abra um terminal dentro da pasta que contém este `README.md` e a pasta `configs`.
4. Confira o conteúdo que será publicado e utilize:

```bash
git init
git branch -M main
git add .
git status
git commit -m "Documenta laboratorio pfSense HA e redundancia de switches"
git remote add origin https://github.com/SEU_USUARIO/eve-ng-pfsense-ha-switch-redundancy.git
git push -u origin main
```

Substitua `SEU_USUARIO` e o nome do repositório pela URL correta. Se essa pasta já for um repositório, revise o remoto existente antes de adicionar `origin`. O push depende da autenticação da sua conta e das permissões do repositório.

## Revisão antes de publicar

- Confirmar que o README aparece na raiz, e não dentro de uma segunda pasta desnecessária.
- Conferir as imagens em `docs/evidencias.md` e o diagrama Mermaid no README.
- Manter a distinção entre resultados comprovados e testes pendentes.
- Usar somente as configurações mínimas deste pacote no repositório público; guardar backups integrais e credenciais em local privado.
- Se desejar uma licença de uso, escolher conscientemente uma licença antes de adicionar um arquivo `LICENSE`. Nenhuma licença foi presumida neste pacote.

## Atualizações posteriores

Após novos testes, atualizar `tests/validacao.md`, substituir o estado de pendência somente com evidência e adicionar capturas identificadas. Registrar o tempo de convergência e o fuso horário para comparações futuras.
