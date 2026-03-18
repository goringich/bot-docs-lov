# otlichniy-ulov-docs

Этот репозиторий существует как docs-only рабочее дерево для проекта `otlichniy-ulov`.

## Obsidian links

- [[Отличный улов/Отличный улов]]
- [[Отличный улов/docs/README|Docs index]]
- [[Отличный улов/docs/18-integration-overview|Integration overview]]
- [[Отличный улов/docs/24-integration-docs-map|Integration docs map]]
- [[Отличный улов/docs/changelog/implementation-status|Implementation status]]

Канонический источник файлов:

- `/home/goringich/Desktop/otlichniy-ulov/docs`

Автосинхронизация устроена через `bind mount`, поэтому один и тот же набор файлов виден сразу в трех местах:

- `/home/goringich/Desktop/otlichniy-ulov/docs`
- `/home/goringich/Desktop/Obsidian/Отличный улов/docs`
- `/home/goringich/Desktop/otlichniy-ulov-docs/docs`

Что важно:

- редактировать можно из любого из этих путей, изменения попадут в тот же набор файлов;
- persistent mount-ы зарегистрированы в `/etc/fstab`;
- эта репа должна трекать только `docs/` и этот `README.md`.

Быстрые проверки:

```bash
mountpoint /home/goringich/Desktop/otlichniy-ulov-docs/docs
mountpoint "/home/goringich/Desktop/Obsidian/Отличный улов/docs"
stat -c '%d:%i %n' \
  /home/goringich/Desktop/otlichniy-ulov/docs/README.md \
  /home/goringich/Desktop/otlichniy-ulov-docs/docs/README.md \
  "/home/goringich/Desktop/Obsidian/Отличный улов/docs/README.md"
```
