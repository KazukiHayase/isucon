PROJECT_ROOT:=/home/isucon/webapp
BUILD_DIR:=go
# BIN_NAME:=
BRANCH:=main

NGX_LOG:=/tmp/access.log
MYSQL_LOG:=/tmp/slow-query.log

DB_HOST:=127.0.0.1
DB_PORT:=3306
DB_USER:=isucon
DB_PASS:=isucon
DB_NAME:=isuports
MYSQL_CMD:=mysql -h$(DB_HOST) -P$(DB_PORT) -u$(DB_USER) -p$(DB_PASS) $(DB_NAME)

ALPSORT=sum
ALPM="/api/organizer/players/[0-9a-zA-Z]+/disqualified,/api/organizer/competition/[0-9a-zA-Z]+/finish,/api/organizer/competition/[0-9a-zA-Z]+/score,/api/player/player/[0-9a-zA-Z]+,/api/player/competition/[0-9a-zA-Z]+/ranking"
OUTFORMAT=count,method,uri,min,max,sum,avg,p99

CHANNEL:=C063N2J4W65

# デプロイ
# make deploy BRANCH=<ブランチ名>でブランチを指定してデプロイ
.PHONY: deploy1
deploy1:
	make before
	make checkout
	make build
	sudo systemctl restart nginx
	sudo systemctl restart isuports
	sudo systemctl disable mysql
	sudo systemctl stop mysql

.PHONY: deploy2
deploy2:
	make before
	make checkout
	sudo systemctl disable nginx
	sudo systemctl stop nginx
	sudo systemctl disable isuports
	sudo systemctl stop isuports
	sudo systemctl restart mysql

.PHONY: deploy3
deploy3:
	make before
	make checkout
	sudo systemctl disable nginx
	sudo systemctl stop nginx
	sudo systemctl disable isuports
	sudo systemctl stop isuports
	sudo systemctl restart mysql

.PHONY: before
before:
	$(eval when := $(shell date "+%s"))
	mkdir -p ~/logs/$(when)
	@if [ -f $(NGX_LOG) ]; then \
		sudo mv -f $(NGX_LOG) ~/logs/$(when)/ ; \
	fi
	@if [ -f $(MYSQL_LOG) ]; then \
		sudo mv -f $(MYSQL_LOG) ~/logs/$(when)/ ; \
	fi

.PHONY: checkout
checkout:
	git fetch && \
	git reset --hard origin/$(BRANCH)

build:
	cd $(BUILD_DIR); \
	make isuports

.PHONY: restart
restart:
	sudo systemctl restart nginx
	sudo systemctl restart mysql
	sudo systemctl restart isuports

# モニタリング
.PHONY: notify
notify: alp slow pprof

## alp
.PHONY: alp
alp:
	sudo alp ltsv --file=$(NGX_LOG) --sort $(ALPSORT) --reverse -o $(OUTFORMAT) -m $(ALPM) | $(SLACKCAT)

## slow-query
.PHONY: slow
slow:
	sudo mysqldumpslow -s t $(MYSQL_LOG) | head -n 20 | $(SLACKCAT)

## pprof
.PHONY: pprof-up
pprof-up:
	go tool pprof -png -output pprof.png http://localhost:6060/debug/pprof/profile?seconds=60

.PHONY: pprof
pprof:
	$(SLACKRAW) -n pprof.png ./pprof.png

# DB
.PHONY: slow-on
slow-on:
	echo "set global slow_query_log_file = '$(MYSQL_LOG)'; set global long_query_time = 0; set global slow_query_log = ON;" | sudo mysql -uroot

.PHONY: slow-off
slow-off:
	echo "set global slow_query_log = OFF;" | sudo mysql -uroot

.PHONY: sql
sql:
	sudo $(MYSQL_CMD)

# Slack通知
.PHONY: slack
slack:
	curl -F file=@$(filepath) -F channels=$(CHANNEL) -H "$(shell cat headers.txt)" https://slack.com/api/files.upload

# セットアップ
.PHONY: init
init:
	sudo apt install -y percona-toolkit dstat git unzip snapd gh graphviz
	wget https://github.com/tkuchiki/alp/releases/download/v1.0.10/alp_linux_amd64.zip
	unzip alp_linux_amd64.zip
	sudo install ./alp /usr/local/bin
	rm alp alp_linux_amd64.zip
