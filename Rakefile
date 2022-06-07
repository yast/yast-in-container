require "yast/rake"

Yast::Tasks.configuration do |conf|
  # lets ignore license check for now
  conf.skip_license_check << /.*/
  conf.install_locations["src/scripts/*"] = File.join(Packaging::Configuration::DESTDIR, "/usr/sbin/")
  conf.install_locations["desktop/*.desktop"] = File.join(
    Packaging::Configuration::DESTDIR, "/usr/share/applications")
  conf.install_locations["README.md"] = conf.install_doc_dir
  conf.install_locations["COPYING"] = conf.install_doc_dir
end
