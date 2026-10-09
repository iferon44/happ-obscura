# «Проверь, как работает NZB» — отчёт по факту проверки

Ветка: `cline/4ax2etqq` (рабочая ветка задачи, не для мержа в `main`).

## TL;DR

В этом репозитории/проекте **никакого NZB нет** — ни файла, ни модуля, ни функции, ни упоминания.
Проверял: содержимое workspace, историю git, апстрим `Happ-proxy` (код, issues, releases), а также
единственный доступный артефакт приложения — сборку Happ 4.2.1 linux x64 (79 MB deb, 1634 файла).
Строка `nzb` встречается в ней **один раз** и это 3 случайных байта внутри сжатого блоба (см. evidence).
Т.е. «проверить, как работает NZB» здесь невозможно: NZB не является частью проекта Happ.
Нужно уточнение от постановщика задачи (варианты — в конце).

## Что это за проект

* Workspace — форк `Happ-proxy/happ-desktop` (`parent`/`source` подтверждены через API), файлов всего два:
  `README.md` и `release`. Кода приложения в репозитории нет (он закрытый).
* `release` — **манифест канала обновлений** клиента Happ. Клиент тянет его по трём URL
  (строки найдены в бинарнике `opt/happ/bin/Happ`):
  * `https://cdn.jsdelivr.net/gh/Happ-proxy/happ-desktop@main/release`
  * `https://cdn.statically.io/gh/Happ-proxy/happ-desktop/main/release`
  * `https://raw.githubusercontent.com/Happ-proxy/happ-desktop/refs/heads/main/release`
* Схема манифеста: `app`, `filenum`, `linux|macos|windows.{block[], beta{}, stable{}}`;
  внутри `beta`/`stable`: `version`, `link`, `details`, `update.{warnalways, updateonly}`.
  `release` в этом форке задаёт только версию/ссылки/блокировки версий — больше ничего.
* Само приложение (по содержимому deb): Qt6/QML GUI + Xray-core (`/opt/happ/bin/core/xray`, `geosite.dat`,
  `geoip.dat`, `routing/`), плюс `happd`, `happ-diag`, `happ-tcping`, `tun/`, `tun2/`, `antifilter/`.

## Evidence (что именно проверено)

1. `grep -ril nzb` по workspace (без `.git`) — 0 совпадений.
2. `git log --all --oneline` — единственный коммит `bd22000 "release 4.2.1"`; никаких следов NZB в истории.
3. GitHub: `search/code?q=nzb+org:Happ-proxy` → 0, `search/issues?q=NZB+org:Happ-proxy` → 0,
   `gh search code/issues --owner Happ-proxy NZB` → пусто. В ветках форка только `main`.
4. Скачал `Happ.linux.x64.deb` 4.2.1 (79479032 B, `Version: 4.2.1-317`), распаковал, прогнал
   поиск по токенам `nzb|usenet|nntp|sabnzbd|newznab|nzbget` по всем 1634 файлам:
   * единственное совпадение — `opt/happ/bin/Happ`, контекст байтов
     `... F4 EE 7B 4E 7A 62 3B 6E 48 DE AD ...` (`{Nzb;nH`) внутри энтропийного блока → совпадение случайное;
   * в `core/xray`, `happd`, `happ-diag`, `happ-tcping` и во всех библиотеках — 0.
5. Набор протоколов, реально присутствующих в бинарнике Happ (строки): `vless`, `vmess`, `trojan`,
   `shadowsocks`, `socks5`, `reality`, `xtls`, `xhttp`, `hysteria`/`hysteria2`, `wireguard`.
   Протокола/режима «NZB» среди них нет.

## Как работает то, что в этом репозитории действительно работает (манифест `release`)

Клиент периодически скачивает JSON с CDN/GitHub, сравнивает `version` своей сборки с `stable`/`beta`
для своей ОС и архитектуры (`x64`/`arm64`), и либо предлагает/выполняет обновление по `link`, либо
блокирует запуск, если текущая версия попала в `block[]` (`warnalways`/`updateonly` — порог
предупреждения/принудительного обновления). Проверено на живых данных:

* 3 URL манифеста из бинарника → HTTP 200 (отдают текущий `main`-манифест).
* Ссылки из локального манифеста живы: `setup-Happ.x64.exe` и `setup-Happ.arm64.exe` 4.2.1 → HTTP 206.
* Замечание: `HEAD`-запросы к `github.com/.../releases/download/...` в этом окружении отдают 401
  (артефакт egress-прокси), обычный GET работает — для проверок используйте GET.

## Побочная находка (может быть важно)

Локальный манифест в форке **отстаёт от апстрима**:

| | локально (bd22000) | upstream `main` |
|---|---|---|
| `filenum` | 50 | 51 |
| `windows.stable.version` | 4.2.1 | 4.3.0 |
| `windows.block[]` | нет `"2.5.2"` | есть `"2.5.2"` |
| `details` | «Improved Xray TUN is now the default…» | «replacing the single global User-Agent option» |

Дополнительно: последний релиз на GitHub у `happ-desktop` уже `4.4.8`, т.е. манифест в апстримном
`main` сам отстаёт от страницы Releases (клиенты, судя по всему, узнают о версиях не только из манифеста).

## Что такое NZB (если имелся в виду Usenet)

NZB — XML-индекс для скачивания бинарных файлов из Usenet: перечисляет сегменты `message-id` по
группам новостей. Клиент (SABnzbd, NZBGet, nzbdav) забирает сегменты по NNTP, декодирует yEnc и
починяет повреждения через PAR2. К прокси-приложению Happ это отношения не имеет — с точки зрения
Happ это просто обычный трафик, который может идти через Xray/TUN.

## Чего не хватает для продолжения

Нужен ответ на уточняющий вопрос (ни один из вариантов не выводится из репозитория):

* **(a)** Usenet NZB — что именно проверять (маршрутизацию такого трафика через прокси/TUN, скорость,
  совместимость с конкретным downloader-ом)?
* **(b)** Опечатка/другое сокращение — что имелось в виду (`XHTTP`, `XTLS`, `TUN`, `NTP` — в бинарнике
  есть сервисы `ntp2-sync.com/v1install`, `time-ntp.com/v1provider`)?
* **(c)** Внутреннее название в другом репозитории/ветке/PR — пришлите ссылку или имя ветки.
