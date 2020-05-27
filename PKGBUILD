# Maintainer: Sven-Hendrik Haase <svenstaro@gmail.com>
# Contributor: Pavol (Lopo) Hluchy <lopo AT losys DOT eu>
# Contributor: Jonas Heinrich <onny@project-insanity.org>
# Contributor: Massimiliano Torromeo <massimiliano.torromeo@gmail.com>
# Contributor: Tobias Hunger <tobias DOT hunger AT gmail DOT com>
# Contributor: Stefan Tatschner <stefan@sevenbyte.org>
# Contributor: Caleb Maclennan <caleb@alerque.com>

# NOTE: Gitlab isn't always compatible with modern Ruby versions. In that case, check the
# commit log for an old fix on how to tell it to use older versions of Ruby. I'm afraid we'll
# need this again at some point in the future.
pkgname=gitlab
pkgver=13.0.1
pkgrel=1
pkgdesc="Project management and code hosting application"
arch=('x86_64')
url="https://gitlab.com/gitlab-org/gitlab-foss"
license=('MIT')
options=(!buildflags)
depends=('ruby' 'ruby-bundler' 'git' 'gitlab-workhorse' 'gitlab-gitaly' 'openssh' 'redis' 'libxslt' 'icu' 're2' 'http-parser' 'nodejs' 'openssl')
makedepends=('cmake' 'postgresql' 'yarn' 'go' 'nodejs')
optdepends=('postgresql: database backend'
            'python-docutils: reStructuredText markup language support'
            'smtp-server: mail server in order to receive mail notifications')
backup=("etc/webapps/gitlab/application.rb"
        "etc/webapps/gitlab/database.yml"
        "etc/webapps/gitlab/gitlab.yml"
        "etc/webapps/gitlab/resque.yml"
        "etc/webapps/gitlab/puma.rb"
        "etc/logrotate.d/gitlab")
source=(git+https://gitlab.com/gitlab-org/gitlab-foss.git#tag=v$pkgver
        configs.patch
        build_fix.patch
        gitlab-puma.service
        gitlab-sidekiq.service
        gitlab-backup.service
        gitlab-mailroom.service
        gitlab-backup.timer
        gitlab.target
        gitlab.tmpfiles.d
        gitlab.logrotate
        ruby27-pop-extra-arg.patch)
install='gitlab.install'
sha512sums=('SKIP'
            '9b054872e2017dae3acd0534c0608634cf7c5f996672e589c3b9988ce18b110423b63f5207585c2ba4941b516606a2a9a8db6fd320012a4d90cf3beca147a220'
            '9623de113358d3d6e49047f688e272d9394579734ace1bd647497e8717a90784546d27e547a29197a16c80d72ad9f2c79eb65f8edc631deadf2ec90ee86ea44b'
            'af5c9d2247e7c1b340865253a3ba4134ee889f11733d055f758c186d9898c78b3d6877a61b88eb31e96670419d7acf21c758c281ab7d7022f81e17388032b9f9'
            'cbaeea4ac8bb5a882eb8a75715ed1d7ac073b2b65074f9b3fbd31aaa10cce72e2580e888a34a40b39014eeaf2ea4fdf74e9589a44149d0d92bc8b55c468dfe07'
            '0cbb9a1631b529a83d5c6db95fd3a684c8f06073890b31f6262c339360444e7452275d804fb6a119a3d61a0ef1b76d0e956f260a12f032d54c00308e8d9520b0'
            '15de5b11a31d733bd5b6fa50faa2395dbe53c252bd52f937e67cdc940de17554e946d1e7f9746538a6be0cc12024fc2816c2b64a56e16762abaca75562a7512d'
            'c76d634647336aaf157bc66ba094a363e971c0d275875a7df4521819147f54cd4c709eb8e024cdac9e900d99167e8a78a222587e7292e915573ef29060e6ec21'
            '879be339148123e32b58a5669fdd3d3bb8b5d711326cb618f95b1680a6ac3a83c85d8862f2691b352fa26c95e4764dbb827856e22a3e2b9e4a76c13fe42864b5'
            'abacbff0d7be918337a17b56481c84e6bf3eddd9551efe78ba9fb74337179e95c9b60f41c49f275e05074a4074a616be36fa208a48fc12d5b940f0554fbd89c3'
            '88e199d2f63e4f235930c35c6dfde80e6010e590907bd4de0af1fbfe6d5491ff56845aefcfe8edefa707712bd84fef96880655747b8bfb949ceeadc0456b0121'
            '0cc5c1df3cd18978df9a01bb64680d3a375c1ff4de6a453045dd26355777b4f08e3a05f55f035c8012a9683100de0bc3d11c280debcb343eb7167fc25342d5c0')


_datadir="/usr/share/webapps/gitlab"
_etcdir="/etc/webapps/gitlab"
_homedir="/var/lib/gitlab"
_logdir="/var/log/gitlab"

prepare() {
  cd gitlab-foss

  # GitLab tries to read its revision information from a file.
  git rev-parse --short HEAD > REVISION

  patch -p1 < ../build_fix.patch
  patch -p1 < ../configs.patch
  # '/home/git' path in the config files indicates a default path that need to be adjusted
  grep -FqR '/home/git' config || exit 1

  cp config/gitlab.yml.example config/gitlab.yml
  cp config/database.yml.postgresql config/database.yml
  cp config/puma.rb.example config/puma.rb
  cp config/resque.yml.example config/resque.yml

  echo "Setting up systemd service files ..."
  for service_file in gitlab-sidekiq.service gitlab-puma.service gitlab.logrotate gitlab-backup.service gitlab-mailroom.service; do
    sed -i "s|<HOMEDIR>|${_homedir}|g" "${srcdir}/${service_file}"
    sed -i "s|<DATADIR>|${_datadir}|g" "${srcdir}/${service_file}"
    sed -i "s|<LOGDIR>|${_logdir}|g" "${srcdir}/${service_file}"
  done

  # https://github.com/bundler/bundler/issues/6882
  sed -e '/BUNDLED WITH/,+1d' -i Gemfile.lock
  bundle lock --update=bundler-audit
  # 'lock' adds 'BUNDLED WITH' back. Remove it again.
  sed -e '/BUNDLED WITH/,+1d' -i Gemfile.lock
}

build() {
  cd gitlab-foss

  echo "Fetching bundled gems..."
  # Gems will be installed into vendor/bundle
  bundle config build.gpgme --use-system-libraries  # See https://bugs.archlinux.org/task/63654
  bundle config force_ruby_platform true # some native gems are not available for newer ruby
  bundle install --jobs=$(nproc) --no-cache --deployment --without development test aws kerberos

  # workaround for a ruby2.7 issue
  # https://gitlab.com/groups/gitlab-org/-/epics/2380
  # https://github.com/ruby-grape/grape/issues/1967
  pushd vendor/bundle/ruby/2.7.0/gems/grape-1.1.0/
  patch -p1 < $srcdir/ruby27-pop-extra-arg.patch
  popd

  yarn install --production --pure-lockfile
  bundle exec rake gitlab:assets:compile RAILS_ENV=production NODE_ENV=production NODE_OPTIONS="--max_old_space_size=4096"
  bundle exec rake gettext:compile RAILS_ENV=production
}

package() {
  depends+=('gitlab-shell')

  cd gitlab-foss

  install -d "${pkgdir}/usr/share/webapps"

  cp -r "${srcdir}"/gitlab-foss "${pkgdir}${_datadir}"
  # Remove unneeded directories: node_modules is only needed during build
  rm -r "${pkgdir}${_datadir}/node_modules"
  # https://gitlab.com/gitlab-org/omnibus-gitlab/blob/194cf8f12e51c26980c09de6388bbd08409e1209/config/software/gitlab-rails.rb#L179
  for dir in spec qa rubocop app/assets vendor/assets; do
    rm -r "${pkgdir}${_datadir}/${dir}"
  done

  chown -R root:root "${pkgdir}${_datadir}"
  chmod 755 "${pkgdir}${_datadir}"

  install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}"
  install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}/satellites"
  install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}/shared/"{,artifacts,lfs-objects}
  install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}/builds"
  install -dm700 -o 105 -g 105 "${pkgdir}${_homedir}/uploads"
  install -dm750 -o 105 -g 105 "${pkgdir}${_homedir}/backups"
  install -dm750 -o 105 -g 105 "${pkgdir}${_etcdir}"
  install -dm755 "${pkgdir}/usr/share/doc/gitlab"

  rm -r "${pkgdir}${_datadir}"/{builds,tmp,log}

  # TODO: Rails uses log dir under the rails root. Figure out if it is possible to configure rails
  # to log right to /var/log/gitlab
  ln -fs "${_logdir}" "${pkgdir}${_datadir}/log"

  # Fixes https://bugs.archlinux.org/task/59762
  ln -s "${_datadir}/config/boot.rb" "${pkgdir}"/${_etcdir}/boot.rb

  # TODO: workhorse and shell secret files are the application data and should be stored under /var/lib/gitlab
  mv "${pkgdir}${_datadir}/.gitlab_workhorse_secret" "${pkgdir}${_etcdir}/gitlab_workhorse_secret"
  chmod 660 "${pkgdir}${_etcdir}/gitlab_workhorse_secret"
  chown root:105 "${pkgdir}${_etcdir}/gitlab_workhorse_secret"
  ln -fs "${_etcdir}/gitlab_workhorse_secret" "${pkgdir}${_datadir}/.gitlab_workhorse_secret"

  ln -fs /etc/webapps/gitlab-shell/secret "${pkgdir}${_datadir}/.gitlab_shell_secret"

  sed -i "s|require_relative '../lib|require '${_datadir}/lib|" config/application.rb

  # Install config files
  for config_file in application.rb gitlab.yml database.yml puma.rb resque.yml; do
    mv "config/${config_file}" "${pkgdir}${_etcdir}/"
    # TODO: configure rails app to use configs right from /etc
    ln -fs "${_etcdir}/${config_file}" "${pkgdir}${_datadir}/config/"
  done

  # Install secrets symlink
  # TODO: ruby uses _datadir to load config files. Figure out if we can load files directly from /etc
  ln -fs "${_etcdir}/secrets.yml" "${pkgdir}${_datadir}/config/secrets.yml"

  # Install license and help files
  mv README.md MAINTENANCE.md CONTRIBUTING.md CHANGELOG.md PROCESS.md VERSION config/*.{example,postgresql} "${pkgdir}/usr/share/doc/gitlab"
  install -Dm644 "LICENSE" "${pkgdir}/usr/share/licenses/gitlab/LICENSE"

  # TODO: structure.sql looks more like an application data and should be stored under /var/lib/gitlab
  chown 105:105 "${pkgdir}${_datadir}/db/structure.sql"

  # Install systemd service files
  for service_file in gitlab-puma.service gitlab-sidekiq.service gitlab-backup.service gitlab-backup.timer gitlab.target gitlab-mailroom.service; do
    install -Dm644 "${srcdir}/${service_file}" "${pkgdir}/usr/lib/systemd/system/${service_file}"
  done

  install -Dm644 "${srcdir}/gitlab.tmpfiles.d" "${pkgdir}/usr/lib/tmpfiles.d/gitlab.conf"
  install -Dm644 "${srcdir}/gitlab.logrotate" "${pkgdir}/etc/logrotate.d/gitlab"
}
