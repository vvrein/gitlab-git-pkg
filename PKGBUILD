# Maintainer: Sven-Hendrik Haase <sh@lutzhaase.com>
# Contributor: Pavol (Lopo) Hluchy <lopo AT losys DOT eu>
# Contributor: Jonas Heinrich <onny@project-insanity.org>
# Contributor: Massimiliano Torromeo <massimiliano.torromeo@gmail.com>
# Contributor: Tobias Hunger <tobias DOT hunger AT gmail DOT com>
# Contributor: Stefan Tatschner <stefan@sevenbyte.org>
# Contributor: Caleb Maclennan <caleb@alerque.com>

pkgname=gitlab
pkgver=8.9.3
pkgrel=1
pkgdesc="Project management and code hosting application"
arch=('i686' 'x86_64')
url="https://gitlab.com/gitlab-org/gitlab-ce/tree/master#README"
license=('MIT')
depends=('ruby2.1' 'git' 'ruby2.1-bundler' 'gitlab-shell' 'gitlab-workhorse' 'openssh' 'redis' 'libxslt' 'icu' 'nodejs')
makedepends=('cmake' 'postgresql' 'mariadb')
optdepends=('postgresql: database backend'
            'mysql: database backend'
            'python2-docutils: reStructuredText markup language support'
            'smtp-server: mail server in order to receive mail notifications')
backup=("etc/webapps/${pkgname}/application.rb"
        "etc/webapps/${pkgname}/gitlab.yml"
        "etc/webapps/${pkgname}/secret"
        "etc/webapps/${pkgname}/resque.yml"
        "etc/webapps/${pkgname}/unicorn.rb"
        "etc/logrotate.d/${pkgname}")
source=("$pkgname-$pkgver.tar.gz::https://github.com/gitlabhq/gitlabhq/archive/v${pkgver}.tar.gz"
        gitlab-unicorn.service
        gitlab-sidekiq.service
        gitlab-backup.service
        gitlab-mailroom.service
        gitlab-backup.timer
        gitlab.target
        gitlab.tmpfiles.d
        gitlab.logrotate
        apache.conf.example
        apache-ssl.conf.example
        apache2.2.conf.example
        apache2.2-ssl.conf.example
        nginx.conf.example
        nginx-ssl.conf.example
        lighttpd.conf.example)
install='gitlab.install'
sha256sums=('39578df8b98113bbe79f49d1435d9c399729e9b0d42062a60c8d8c11d1fe056b'
            'becafe0f9811fea69a69b8e2739857ef007f0b7e89391229f123c79c285f34f3'
            'fbe5ec709ead1729e4de85f3f036f053b2b14041c540742315ff2d63a7bdd59a'
            'd21d8c961b2834115a1d9c646278782aaaae0d1d1cde2357b58e67bad3a58527'
            '06c9f6575ddaeb9cfb70dd9c6cc50a2e676b0aba42731d1c7793a5ba12a2d4e5'
            'e2539301fe42869d8fdbaa1b53b30076fb436c4220a37e576ed704458f804852'
            'a1ee236a1f3e65cd26d9adb5f636f66fbab68777fd60d1c796cb26036bd0903f'
            '84614a2bfbd734f09c2c91531dd3c13e795186b50c0780a120c8e5bc2a892607'
            '7b3137b8175db06e97c7577fb1df3d9095ff0797e6428c12d9c633ddd9121ad5'
            '87fa65bc2d8f382d22fe77a6958bac9058e99021b230e2922a5b7e7afff39dcc'
            '5e8c0e5d66ae5039620bd5d92076112bd47d9894a9cfbf06242dad412618f01a'
            'b4b10b401de60a714ebb38b0e17c9efe123967565d9b73297503fbaea4bcf03d'
            '8944a5eb8972a63f962dc34ed1c2843e019b2b521d8f045a2552ddc2f2e28ec3'
            '481427bec661c8bebc652a3349e10dd8c9435f51a0dcbb7b2e6833309ce90a1b'
            '822d0b80f1974c8418a9f4d66fbefb7679313b6de9a49c137c83c0bfe622460f'
            'ea5a5f0b4c0ffd26d977efaf564800ee7fa88579a9e4f0556143a591a7ff198c')

_homedir="/var/lib/${pkgname}"
_datadir="/usr/share/webapps/${pkgname}"
_logdir="/var/log/${pkgname}"
_srcdir="gitlabhq-${pkgver}"
_etcdir="/etc/webapps/${pkgname}"

prepare() {
  cd "${srcdir}/${_srcdir}"

  # Patching config files:
  msg2 "Patching paths in and username gitlab.yml..."
  sed -e "s|# user: git|user: gitlab|" \
      -e "s|/home/git/repositories|${_homedir}/repositories|" \
      -e "s|/home/git/gitlab-satellites|${_homedir}/satellites|" \
      -e "s|# path: /mnt/gitlab|path: ${_homedir}/shared|" \
      -e "s|/home/git/gitlab-shell|/usr/share/webapps/gitlab-shell|" \
      -e "s|tmp/backups|${_homedir}/backups|" \
      config/gitlab.yml.example > config/gitlab.yml

  msg2 "Patching paths and timeout in unicorn.rb..."
  sed -e "s|/home/git/gitlab/tmp/.*/|/run/gitlab/|g" \
      -e "s|/var/run/|/run/|g" \
      -e "s|/home/git/gitlab|${_datadir}|g" \
      -e "s|timeout 30|timeout 300|" \
      -e "s|${_datadir}/log/|${_logdir}/|g" \
      config/unicorn.rb.example > config/unicorn.rb

  # We need this one untouched because otherwise assets will fail
  cp config/database.yml.postgresql config/database.yml.postgresql.orig

  msg2 "Patching username in database.yml.{mysql,postgresql}..."
  sed -i -e "s|username: git|username: gitlab|" config/database.yml.mysql
  sed -i -e "s|username: git|username: gitlab|" config/database.yml.postgresql

  msg2 "Patching redis connection in resque.yml"
  sed -e "s|production: unix:/var/run/redis/redis.sock|production: redis://localhost:6379|" \
      config/resque.yml.example > config/resque.yml

  msg2 "Setting up systemd service files ..."
  for service_file in gitlab-sidekiq.service gitlab-unicorn.service gitlab.logrotate gitlab-backup.service gitlab-mailroom.service; do
    sed -i "s|<HOMEDIR>|${_homedir}|g" "${srcdir}/${service_file}"
    sed -i "s|<DATADIR>|${_datadir}|g" "${srcdir}/${service_file}"
    sed -i "s|<LOGDIR>|${_logdir}|g" "${srcdir}/${service_file}"
  done
}

build() {
  cd "${srcdir}/${_srcdir}"

  msg "Fetching bundled gems..."
  # Gems will be installed into vendor/bundle

  bundle-2.1 config build.nokogiri --use-system-libraries
  bundle-2.1 install -j$(nproc) --no-cache --deployment --without development test aws kerberos

  # We'll temporarily stick this in here so we can build the assets
  cp config/database.yml.postgresql.orig config/database.yml
  sed -i '/symlink/d' config/initializers/gitlab_shell_secret_token.rb
  bundle-2.1 exec rake assets:precompile RAILS_ENV=production
  # After building assets, clean this up again
  rm config/database.yml config/database.yml.postgresql.orig
}

package() {
  cd "${srcdir}/${_srcdir}"

  install -d "${pkgdir}/usr/share/webapps"

  cp -r "${srcdir}/${_srcdir}" "${pkgdir}${_datadir}"
  chown -R 105:105 "${pkgdir}${_datadir}"
  chmod 755 "${pkgdir}${_datadir}"

  install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}"
  install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}/satellites"
  install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}/shared"
  install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}/builds"
  install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}/uploads"
  install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}/backups"
  install -dm750 -o 105 -g 105 "${pkgdir}${_etcdir}"
  install -dm755 "${pkgdir}/usr/share/doc/${pkgname}"

  touch "${pkgdir}${_etcdir}/secret"
  chmod 640 "${pkgdir}${_etcdir}/secret"
  chown root:105 "${pkgdir}${_etcdir}/secret"

  ln -fs /run/gitlab "${pkgdir}${_homedir}/pids"
  ln -fs /run/gitlab "${pkgdir}${_homedir}/sockets"
  ln -fs ${_datadir}/log "${pkgdir}${_homedir}/log"
  ln -fs "${_etcdir}/secret" "${pkgdir}${_datadir}/.secret"

  rm -rf "${pkgdir}${_datadir}/public/uploads" && ln -fs "${_homedir}/uploads" "${pkgdir}${_datadir}/public/uploads"
  rm -rf "${pkgdir}${_datadir}/builds" && ln -fs "${_homedir}/builds" "${pkgdir}${_datadir}/builds"
  rm -rf "${pkgdir}${_datadir}/tmp" && ln -fs /var/tmp "${pkgdir}${_datadir}/tmp"
  rm -rf "${pkgdir}${_datadir}/log" && ln -fs "${_logdir}" "${pkgdir}${_datadir}/log"

  ln -fs /etc/webapps/gitlab-shell/secret "${pkgdir}${_datadir}/.gitlab_shell_secret"

  sed -i "s|require_relative '../lib|require '${_datadir}/lib|" config/application.rb

  # Fix for ruby-2.1 and bundle-2.1
  sed -i "s|bundle|bundle-2.1|g" "${pkgdir}${_datadir}/lib/tasks/gitlab/check.rake"
  grep -rl "bin/env ruby" "${pkgdir}${_datadir}" | xargs sed -i "s|bin/env ruby$|bin/env ruby-2.1|g"
  
  # Install config files
  for config_file in application.rb gitlab.yml unicorn.rb resque.yml; do
    mv "config/${config_file}" "${pkgdir}${_etcdir}/"
    [[ -f "${pkgdir}${_datadir}/config/${config_file}" ]] && rm "${pkgdir}${_datadir}/config/${config_file}"
    ln -fs "${_etcdir}/${config_file}" "${pkgdir}${_datadir}/config/"
  done

  # Install database symlink
  ln -fs "${_etcdir}/database.yml" "${pkgdir}${_datadir}/config/database.yml"

  # Install secrets symlink
  ln -fs "${_etcdir}/secrets.yml" "${pkgdir}${_datadir}/config/secrets.yml"

  # Install license and help files
  mv README.md MAINTENANCE.md CONTRIBUTING.md CHANGELOG config/*.{example,mysql,postgresql} "${pkgdir}/usr/share/doc/${pkgname}"
  install -D "LICENSE" "${pkgdir}/usr/share/licenses/${pkgname}/LICENSE"

  # https://gitlab.com/gitlab-org/gitlab-ce/issues/765
  cp -r "${pkgdir}${_datadir}/doc" "${pkgdir}${_datadir}/public/help"
  find "${pkgdir}${_datadir}/public/help" -name "*.md" -exec rm {} \;
  find "${pkgdir}${_datadir}/public/help/" -depth -type d -empty -exec rmdir {} \;

  # Install systemd service files
  for service_file in gitlab-unicorn.service gitlab-sidekiq.service gitlab-backup.service gitlab-backup.timer gitlab.target gitlab-mailroom.service; do
    install -Dm644 "${srcdir}/${service_file}" "${pkgdir}/usr/lib/systemd/system/${service_file}"
  done

  install -Dm644 "${srcdir}/gitlab.tmpfiles.d" "${pkgdir}/usr/lib/tmpfiles.d/gitlab.conf"
  install -Dm644 "${srcdir}/gitlab.logrotate" "${pkgdir}/etc/logrotate.d/gitlab"

  # Install webserver config templates
  for config_file in apache apache-ssl apache2.2 apache2.2-ssl nginx nginx-ssl lighttpd; do
    install -m644 "${srcdir}/${config_file}.conf.example" "${pkgdir}/usr/share/doc/${pkgname}"
  done
}

# vim:set ts=2 sw=2 et:
