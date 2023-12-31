# Maintainer: Anatol Pomozov <anatol.pomozov@gmail.com>
# Maintainer: Caleb Maclennan <caleb@alerque.com>
# Contributor: Sven-Hendrik Haase <svenstaro@gmail.com>
# Contributor: Pavol (Lopo) Hluchy <lopo AT losys DOT eu>
# Contributor: Jonas Heinrich <onny@project-insanity.org>
# Contributor: Massimiliano Torromeo <massimiliano.torromeo@gmail.com>
# Contributor: Tobias Hunger <tobias DOT hunger AT gmail DOT com>
# Contributor: Stefan Tatschner <stefan@sevenbyte.org>
# Contributor: loqs <bugs-archlinux@entropy-collector.net>

pkgname=gitlab
pkgver=16.2.3
pkgrel=1
pkgdesc='Project management and code hosting application'
arch=(x86_64)
url='https://gitlab.com/gitlab-org/gitlab-foss'
license=(MIT)
depends=(git
         gitlab-gitaly
         http-parser
         icu libicui18n.so libicuuc.so
         libxslt
         nodejs
         openssh
         openssl
         perl-image-exiftool
         re2
         redis
         ruby)
makedepends=(cmake
             go
             nodejs
             postgresql
             yarn)
optdepends=('postgresql: database backend'
            'python-docutils: reStructuredText markup language support'
            'smtp-server: mail server in order to receive mail notifications')
backup=("etc/webapps/$pkgname/database.yml"
        "etc/webapps/$pkgname/gitlab.yml"
        "etc/webapps/$pkgname/resque.yml"
        "etc/webapps/$pkgname/puma.rb"
        "etc/webapps/$pkgname/smtp_settings.rb"
        "etc/logrotate.d/$pkgname")
options=(!buildflags !debug)
source=("git+$url.git#tag=v$pkgver"
        "$pkgname-configs.patch"
        "$pkgname-environment"
        "$pkgname-puma.service"
        "$pkgname-sidekiq.service"
        "$pkgname-backup.service"
        "$pkgname-mailroom.service"
        "$pkgname-workhorse.service"
        "$pkgname-backup.timer"
        "$pkgname.target"
        "$pkgname.tmpfiles.d"
        "$pkgname.logrotate"
        "gemfile.patch")
provides=(gitlab-workhorse) # FS78036
conflicts=(gitlab-workhorse)
replaces=(gitlab-workhorse)
install=gitlab.install
sha256sums=('SKIP'
            '87965cd99420b5b9acad08b9cd7aa6e19ad18f81b41e19ee4bd5c8c4506d9681'
            '8cc4d933743906b4213b8ea8d8c5a62535e27e4073f73581a5dad40078dde000'
            '20fb734203ee13530b4c1038dd197a7584657302cb783288f8055059529d80d8'
            '22c9c255552eeb7c3dc457438aaa0b0782172eeb12c3d419b34040a3f1e95eae'
            '91fa21253c32ad4a68fb2bd5051a0d0d2756b4efd3b59cdf16d4bb8e77d3f886'
            '90bbbf991490bfe350a3c18c939377eccd1011c1193f004ef08d0e4435457276'
            '41bb959e861252e130206e170a1f38dc77e270a951062b0513826de253eabc15'
            '869b3e682e9fb26551a19c0cd0b200a6fdb594396f325e237d58e1a8a8a96f73'
            '6c96a5d20c03bd626d9408cb1e41ab131d67610be586475af17c1e52e27ec697'
            '35858f5a4db0ab703e0099dd25f71910b2253e73eb65fdaec89bf5ab64d008e9'
            '13e4588b62ebaa6b410c2192cafbd2b9f2c99b8fff7b02782c2968c8256f762a'
            'SKIP')

_appdir=/usr/share/webapps/gitlab # the app source code location
_etcdir=/etc/webapps/gitlab
_datadir=/var/lib/gitlab # directory with gitlab data and it also $HOME for 'gitlab' user
_logdir=/var/log/gitlab

prepare() {
	cd gitlab-foss

	# GitLab tries to read its revision information from a file.
	git rev-parse --short HEAD > REVISION

	patch -p1 -i ../$pkgname-configs.patch
	patch -p1 -i ../gemfile.patch

	# '/home/git' path in the config files indicates a default path that need to be adjusted
	grep -FqR '/home/git' config || exit 1

	cp config/gitlab.yml.example config/gitlab.yml
	cp config/database.yml.postgresql config/database.yml
	cp config/puma.rb.example config/puma.rb
	cp config/resque.yml.example config/resque.yml
	cp config/initializers/smtp_settings.rb.sample config/initializers/smtp_settings.rb

	echo "Setting up systemd service files ..."
	for service_file in gitlab-sidekiq.service gitlab-puma.service gitlab.logrotate gitlab-backup.service gitlab-mailroom.service; do
		sed -i "s|<DATADIR>|${_datadir}|g" "${srcdir}/${service_file}"
		sed -i "s|<APPDIR>|${_appdir}|g" "${srcdir}/${service_file}"
		sed -i "s|<LOGDIR>|${_logdir}|g" "${srcdir}/${service_file}"
	done
}

build() {
	cd gitlab-foss

	echo "Fetching bundled gems..."
	# Gems will be installed into vendor/bundle

	bundle config set --local deployment 'true'
	bundle config set --local without 'development test aws kerberos'
	BUNDLER_CHECKSUM_VERIFICATION_OPT_IN=1 bundle install --jobs=$(nproc)

	export CGO_CPPFLAGS="${CPPFLAGS}"
	export CGO_CFLAGS="${CFLAGS}"
	export CGO_CXXFLAGS="${CXXFLAGS}"
	export CGO_LDFLAGS="${LDFLAGS}"
	export GOFLAGS="-buildmode=pie -trimpath -mod=readonly -modcacherw"
	make -C workhorse

	yarn install --production --pure-lockfile
	bundle exec rake gettext:compile RAILS_ENV=production NODE_ENV=production USE_DB=false SKIP_STORAGE_VALIDATION=true NODE_OPTIONS="--max_old_space_size=3584"
	bundle exec rake gitlab:assets:compile RAILS_ENV=production NODE_ENV=production USE_DB=false SKIP_STORAGE_VALIDATION=true NODE_OPTIONS="--max_old_space_size=3584"
}

package() {
	depends+=('gitlab-shell')

	cd gitlab-foss

	install -d "${pkgdir}/usr/share/webapps"

	cp -r "${srcdir}"/gitlab-foss "${pkgdir}${_appdir}"
	# Remove unneeded directories: node_modules is only needed during build
	rm -r "${pkgdir}${_appdir}/node_modules"
	# https://gitlab.com/gitlab-org/omnibus-gitlab/blob/194cf8f12e51c26980c09de6388bbd08409e1209/config/software/gitlab-rails.rb#L179
	for dir in spec qa rubocop app/assets vendor/assets; do
		rm -r "${pkgdir}${_appdir}/${dir}"
	done

	chown -R root:root "${pkgdir}${_appdir}"
	chmod 755 "${pkgdir}${_appdir}"

	install -Dm755 "workhorse/gitlab-workhorse" "${pkgdir}/usr/bin/gitlab-workhorse"
	install -Dm755 "workhorse/gitlab-zip-cat" "${pkgdir}/usr/bin/gitlab-zip-cat"
	install -Dm755 "workhorse/gitlab-zip-metadata" "${pkgdir}/usr/bin/gitlab-zip-metadata"

	install -dm750 "${pkgdir}${_datadir}"
	install -dm750 "${pkgdir}${_datadir}/satellites"
	install -dm750 "${pkgdir}${_datadir}/shared/"{,artifacts,lfs-objects}
	install -dm750 "${pkgdir}${_datadir}/builds"
	install -dm700 "${pkgdir}${_datadir}/uploads"
	install -dm750 "${pkgdir}${_datadir}/backups"
	install -dm755 "${pkgdir}${_etcdir}"
	install -dm755 "${pkgdir}${_logdir}"
	install -dm755 "${pkgdir}/usr/share/doc/gitlab"

	rm -r "${pkgdir}${_appdir}"/{.git,builds,tmp,log,shared}

	# Rails app hardcodes/configures by default that data is stored under $_appdir
	# Create symlinks that point to data directories under /var
	ln -fs "${_logdir}" "${pkgdir}${_appdir}/log"
	ln -fs "${_datadir}/builds" "${pkgdir}${_appdir}/builds"
	ln -fs "${_datadir}/uploads" "${pkgdir}${_appdir}/public/uploads"
	ln -fs "${_datadir}/shared" "${pkgdir}${_appdir}/shared"
	# The path to backups is configured in gitlab.yml, but the gitlab:backup rake
	# task writes a PID file in this directory (the path is hardcoded in
	# /usr/share/webapps/gitlab/lib/tasks/gitlab/backup.rake).
	# See https://bugs.archlinux.org/task/76630
	ln -fs /var/tmp "${pkgdir}${_appdir}/tmp"

	# TODO: workhorse and shell secret files are the application data and should be stored under /var/lib/gitlab
	ln -fs "${_etcdir}/gitlab_workhorse_secret" "${pkgdir}${_appdir}/.gitlab_workhorse_secret"
	ln -fs /etc/webapps/gitlab-shell/secret "${pkgdir}${_appdir}/.gitlab_shell_secret"

	# Install config files
	for config_file in gitlab.yml database.yml puma.rb resque.yml; do
		mv "config/${config_file}" "${pkgdir}${_etcdir}/"
		# TODO: configure rails app to use configs right from /etc
		ln -fs "${_etcdir}/${config_file}" "${pkgdir}${_appdir}/config/"
	done
	mv "config/initializers/smtp_settings.rb" "${pkgdir}${_etcdir}/"
	ln -fs "${_etcdir}/smtp_settings.rb" "${pkgdir}${_appdir}/config/initializers/smtp_settings.rb"

	# Install secrets symlink
	# TODO: ruby uses _appdir to load config files. Figure out if we can load files directly from /etc
	ln -fs "${_etcdir}/secrets.yml" "${pkgdir}${_appdir}/config/secrets.yml"

	# files with passwords/secrets are set world-unreadable
	for secret_file in smtp_settings.rb; do
		chmod 660 "${pkgdir}${_etcdir}/${secret_file}"
		# TODO: should we just leave the secret files root owned?
		chown root:root "${pkgdir}${_etcdir}/${secret_file}"
	done

	install -Dm644 "${srcdir}/$pkgname-environment" "${pkgdir}${_appdir}/environment"

	# Install license and help files
	mv README.md MAINTENANCE.md CONTRIBUTING.md CHANGELOG.md PROCESS.md VERSION config/*.{example,postgresql} "${pkgdir}/usr/share/doc/gitlab"
	install -Dm644 "LICENSE" "${pkgdir}/usr/share/licenses/gitlab/LICENSE"

	# TODO: structure.sql looks more like an application data and should be stored under /var/lib/gitlab
	#chown 105:105 "${pkgdir}${_appdir}/db/structure.sql"

	# Install systemd service files
	for service_file in gitlab-puma.service gitlab-sidekiq.service gitlab-backup.service gitlab-backup.timer gitlab.target gitlab-mailroom.service gitlab-workhorse.service; do
		install -Dm644 "${srcdir}/${service_file}" "${pkgdir}/usr/lib/systemd/system/${service_file}"
	done

	install -Dm644 "${srcdir}/gitlab.tmpfiles.d" "${pkgdir}/usr/lib/tmpfiles.d/gitlab.conf"
	install -Dm644 "${srcdir}/gitlab.logrotate" "${pkgdir}/etc/logrotate.d/gitlab"
}
