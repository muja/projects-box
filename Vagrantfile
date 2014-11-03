# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "phusion/ubuntu-14.04-amd64"

  config.vm.hostname = "gitlab.webhippie.de"
  config.vm.network "forwarded_port", guest: 10080, host: 10080
  config.vm.network "forwarded_port", guest: 10022, host: 10022

  config.omnibus.chef_version = :latest

  config.vm.provider :virtualbox do |provider|
    provider.gui = false

    provider.memory = 2048
    provider.cpus = 4
    provider.name = "gitlab"
  end

  config.vm.provision :chef_solo, run: :always do |chef|
    # chef.cookbooks_path = ["custom"]
    chef.log_level = :debug

    chef.add_recipe "docker"
    chef.add_recipe "lwrpexec"

    chef.json = {
      "docker" => {
        "container_init_type" => false
      },
      "lwrpexec" => {
        "run_list" => [
          ["docker_image", "sameersbn/gitlab:latest"],
          ["docker_image", "postgres:latest"],
          ["docker_image", "dockerfile/redis"],

          # REDIS
          ["docker_container", "redis", { image: "dockerfile/redis" }],

          # DATA-CONTAINER for gitlab
          ["docker_container", "gitlab-data", { image: "sameersbn/gitlab", entrypoint: "true" }],

          # DATA-CONTAINER for postgresql
          ["docker_container", "postgres-data", { image: "postgres", entrypoint: "true" }],

          # POSTGRESQL
          ["docker_container", "postgres", { image: "postgres", volumes_from: "postgres-data" }],

          # pre-init a pgsql configuration resource (resource names are unique in chef.)
          [
            "docker_container", "pg - configure gitlab", {
              detach: false,
              has_name: false,
              image: "postgres",
              link: ["postgres:postgres"],
              # wait for pgsql to be ready.
              command: "bash -c 'while ! pg_isready -h postgres > /dev/null 2>&1; do sleep 1; done'"
            }
          ],

          # POSTGRESQL - create GitLabHQ user
          [
            "docker_container", "pg - configure gitlab", {
              command: "bash -c 'createuser -h postgres -U postgres gitlabhq; true'",
            }
          ],

          # POSTGRESQL - create GitLabHQ database
          [
            "docker_container", "pg - configure gitlab", {
              command: "bash -c 'createdb -h postgres -U postgres -O gitlabhq gitlabhq_production; true'",
            }
          ],

          # GITLAB
          [
            "docker_container", "gitlab", {
              image: "sameersbn/gitlab",
              link: [
                "postgres:postgresql",
                "redis:redisio"
              ],
              volumes_from: "gitlab-data",
              env: [
                "DB_USER=gitlabhq",
                "DB_NAME=gitlabhq_production"
              ],
              port: [
                "10080:80",
                "10022:22"
              ]
            }
          ]
        ]
      }
    }
  end
end
