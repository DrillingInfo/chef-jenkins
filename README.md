Description
===========

Installs and configures Jenkins CI server & node slaves.  Resource providers to support automation via jenkins-cli, including job create/update.

Changelog
=========

- 0.7 - Jenkins was binding to the ip address of the primary interface instead of listening to all addresses which caused nginx to be unable to connect over localhost to Jenkins.
- 0.8 - Convert the recipe to use only the WAR file and not the Debian package.

Requirements
============

Chef
----

* Chef version 0.9.10 or higher

Platform
--------

* 'default' - Server installation - currently supports Red Hat/CentOS 5.x and Ubuntu 8.x/9.x/10.x

* 'node_ssh' - Any platform that is running sshd.

* 'node_jnlp' - Unix platforms. (depends on runit recipe)

* 'node_windows' - Windows platforms only.  Depends on .NET Framework, which can be installed with the windows::dotnetfx recipe.

Java
----

Jenkins requires Java 1.5 or higher, which can be installed via the Opscode java cookbook or windows::java recipe.

Jenkins node authentication
---------------------------

If your Jenkins instance requires authentication, you'll either need to embed user:pass in the jenkins.server.url or issue a jenkins-cli.jar login command prior to using the jenkins::node_* recipes.  For example, define a role like so:

  name "jenkins_ssh_node"
  description "cli login & register ssh slave with Jenkins"
  run_list %w(vmw::jenkins_login jenkins::node_ssh)

Where the jenkins_login recipe is simply:

  jenkins_cli "login --username #{node[:jenkins][:username]} --password #{node[:jenkins][:password]}"

Attributes
----------

* jenkins[:version] - Specify the version of jenkins used.  By default it is latest. Note that if the version is 'latest' and jenkins[:server][:lock_version] is `false`, then jenkins will be updated for every stable release.
* jenkins[:war_sha] - The SHA-256 sum of the jenkins war file. If the SHA-256 sum of the currently installed war file matches this sum, it will not be downloaded again. Note that the checksum will not be automatically recorded. Also, if both `version` and `war_sha` are specified, `war_sha` will take precedence; i.e. if you want to upgrade jenkins by explicitly setting a new `version`, you will need to reset or delete `war_sha` as well.
* jenkins[:mirror_url] - Specify the base URL to download the WAR and any plugins from
* jenkins[:java_home] - Java install path, used for for cli commands
* jenkins[:server][:home] - JENKINS_HOME directory
* jenkins[:server][:user] - User the Jenkins server runs as
* jenkins[:server][:prefix] - Prefix for the Jenkins servlet context
* jenkins[:server][:group] - Jenkins user primary group
* jenkins[:server][:port] - TCP listen port for the Jenkins server
* jenkins[:server][:url] - Base URL of the Jenkins server
* jenkins[:server][:plugins] - Download the plugins in this list, bypassing update center. 
* jenkins[:server][:lock_version] - If true and jenkins[:version] is 'latest', then the version and sha-256 of the war file that is installed will be recorded in the jenkins[:version] and jenkins[:war_sha] properties respectively. Doing so will lock the jenkins install to that version until manually modified. By default, this attribute is set to false
* jenkins[:node][:name] - Name of the node within Jenkins
* jenkins[:node][:description] - Jenkins node description
* jenkins[:node][:executors] - Number of node executors
* jenkins[:node][:home] - Home directory ("Remote FS root") of the node
* jenkins[:node][:labels] - Node labels
* jenkins[:node][:mode] - Node usage mode, "normal" or "exclusive" (tied jobs only)
* jenkins[:node][:launcher] - Node launch method, "jnlp", "ssh" or "command"
* jenkins[:node][:availability] - "always" keeps node on-line, "demand" off-lines when idle
* jenkins[:node][:in_demand_delay] - number of minutes for which jobs must be waiting in the queue before attempting to launch this slave.
* jenkins[:node][:idle_delay] - number of minutes that this slave must remain idle before taking it off-line.
* jenkins[:node][:env] - "Node Properties" -> "Environment Variables"
* jenkins[:node][:user] - user the slave runs as
* jenkins[:node][:ssh_host] - Hostname or IP Jenkins should connect to when launching an SSH slave
* jenkins[:node][:ssh_port] - SSH slave port
* jenkins[:node][:ssh_user] - SSH slave user name (only required if jenkins server and slave user is different)
* jenkins[:node][:ssh_pass] - SSH slave password (not required when server is installed via default recipe)
* jenkins[:node][:ssh_private_key] - jenkins master defaults to: `~/.ssh/id_rsa` (created by the default recipe)
* jenkins[:node][:jvm_options] - SSH slave JVM options
* jenkins[:iptables_allow] - if iptables is enabled, add a rule passing 'jenkins[:server][:port]'
* jenkins[:http_proxy][:variant] - use `nginx` or `apache2` to proxy traffic to jenkins backend (`nil` by default)
* jenkins[:http_proxy][:www_redirect] - add a redirect rule for 'www.*' URL requests ("disable" by default)
* jenkins[:http_proxy][:listen_ports] - list of HTTP ports for the HTTP proxy to listen on ([80] by default)
* jenkins[:http_proxy][:host_name] - primary vhost name for the HTTP proxy to respond to (`node[:fqdn]` by default)
* jenkins[:http_proxy][:host_aliases] - optional list of other host aliases to respond to (empty by default)
* jenkins[:http_proxy][:client_max_body_size] - max client upload size ("1024m" by default, nginx only)

Usage
-----

'default' recipe
----------------

Installs a Jenkins CI server using the http://jenkins-ci.org/war-stable WAR.  The recipe also generates an ssh private key and stores the ssh public key in the node 'jenkins[:pubkey]' attribute for use by the node recipes.

This recipe will also install plugins listed in the node`[:jenkins][:plugins]`
attribute.  This attribute should be a hash with the names of the plugins as
keys. The values can either be empty hashes or hashes containing the keys
'version' (the version of the plugin to install, will default to the latest),
and 'download_url' (an explicit download url for the plugin). If either of these
keys are omitted, then they will be recorded after the first run, and will be
used on subsequent runs.


'node_ssh' recipe
-----------------

Creates the user and group for the Jenkins slave to run as and sets `.ssh/authorized_keys` to the 'jenkins[:pubkey]' attribute.  The 'jenkins-cli.jar'[1] is downloaded from the Jenkins server and used to manage the nodes via the 'groovy'[2] cli command.  Jenkins is configured to launch a slave agent on the node using its SSH slave plugin[3].

[1] http://wiki.jenkins-ci.org/display/JENKINS/Jenkins+CLI
[2] http://wiki.jenkins-ci.org/display/JENKINS/Jenkins+Script+Console
[3] http://wiki.jenkins-ci.org/display/JENKINS/SSH+Slaves+plugin

'node_jnlp' recipe
------------------

Creates the user and group for the Jenkins slave to run as and '/jnlpJars/slave.jar' is downloaded from the Jenkins server.  Depends on runit_service from the runit cookbook.

'node_windows' recipe
---------------------

Creates the home directory for the node slave and sets 'JENKINS_HOME' and 'JENKINS_URL' system environment variables.  The 'winsw'[1] Windows service wrapper will be downloaded and installed, along with generating `jenkins-slave.xml` from a template.  Jenkins is configured with the node as a 'jnlp'[2] slave and '/jnlpJars/slave.jar' is downloaded from the Jenkins server.  The 'jenkinsslave' service will be started the first time the recipe is run or if the service is not running.  The 'jenkinsslave' service will be restarted if '/jnlpJars/slave.jar' has changed.  The end results is functionally the same had you chosen the option to "Let Jenkins control this slave as a Windows service"[3].

[1] http://weblogs.java.net/blog/2008/09/29/winsw-windows-service-wrapper-less-restrictive-license
[2] http://wiki.jenkins-ci.org/display/JENKINS/Distributed+builds
[3] http://wiki.jenkins-ci.org/display/JENKINS/Installing+Jenkins+as+a+Windows+service

'proxy_nginx' recipe
--------------------

Uses the nginx::source recipe from the nginx cookbook to install an HTTP frontend proxy. To automatically activate this recipe set the `node[:jenkins][:http_proxy][:variant]` to `nginx`.

'proxy_apache2' recipe
----------------------

Uses the apache2 recipe from the apache2 cookbook to install an HTTP frontend proxy. To automatically activate this recipe set the `node[:jenkins][:http_proxy][:variant]` to `apache2`.

-------------------------------
'jenkins_cli' resource provider

This resource can be used to execute the Jenkins cli from your recipes.  For example, install plugins via update center and restart Jenkins:

    %w(git URLSCM build-publisher).each do |plugin|
      jenkins_cli "install-plugin #{plugin}"
      jenkins_cli "safe-restart"
    end

'jenkins_node' resource provider
--------------------------------

This resource can be used to configure nodes as the 'node_ssh' and 'node_windows' recipes do or "Launch slave via execution of command on the Master".

    jenkins_node node[:fqdn] do
      description  "My node for things, stuff and whatnot"
      executors    5
      remote_fs    "/var/jenkins"
      launcher     "command"
      command      "ssh -i my_key #{node[:fqdn]} java -jar #{remote_fs}/slave.jar"
      env          "ANT_HOME" => "/usr/local/ant", "M2_REPO" => "/dev/null"
    end

'jenkins_job' resource provider
-------------------------------

This resource manages jenkins jobs, supporting the following actions:

   :create, :update, :delete, :build, :disable, :enable

The 'create' and 'update' actions require a jenkins job config.xml.  Example:

    git_branch = 'master'
    job_name = "sigar-#{branch}-#{node[:os]}-#{node[:kernel][:machine]}"

    job_config = File.join(node[:jenkins][:node][:home], "#{job_name}-config.xml")

    jenkins_job job_name do
      action :nothing
      config job_config
    end

    template job_config do
      source "sigar-jenkins-config.xml"
      variables :job_name => job_name, :branch => git_branch, :node => node[:fqdn]
      notifies :update, resources(:jenkins_job => job_name), :immediately
      notifies :build, resources(:jenkins_job => job_name), :immediately
    end

'jenkins_plugin' resource provider
----------------------------------

This resource will install Jenkins plugins, and record their version, sha-256,
and the url they were downloaded from.

    jenkins_plugin do
        name 'git'
        version '1.1.18'
        download_url 'http://mirrors.jenkins-ci.org/plugins/git/1.1.18/git.hpi'
    end

The only required attribute is the name of the plugin. If version is left out,
it will be recorded after the plugin is installed and will be used for future
runs of the provider, i.e. once a plugin is installed, it will not be upgraded
unless the version is explicitly changed.

'manage_node' library
---------------------

The script to generate groovy that manages a node can be used standalone.  For example:

    % ruby manage_node.rb name slave-hostname remote_fs /home/jenkins ... | java -jar jenkins-cli.jar -s http://jenkins:8080/ groovy =

Issues
------

* CLI authentication - http://issues.jenkins-ci.org/browse/JENKINS-3796

* CLI *-node commands fail with "No argument is allowed: nameofslave" - http://issues.jenkins-ci.org/browse/JENKINS-5973

License & Author(s):
-------------------

This is a downstream fork of Doug MacEachern's Hudson cookbook (https://github.com/dougm/site-cookbooks) and therefore deserves all the glory.

Author:: Doug MacEachern (<dougm@vmware.com>)

Contributor:: AJ Christensen <aj@junglist.gen.nz>
Contributor:: Fletcher Nichol <fnichol@nichol.ca>
Contributor:: Roman Kamyk <rkj@go2.pl>
Contributor:: Darko Fabijan <darko@renderedtext.com>
Contributor:: Scott Likens <scott@likens.us>

Copyright:: 2010, VMware, Inc

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
