source 'https://rubygems.org'

gemspec

group :development do
	#We depend on Vagrant for development, but we don't add it as a
	#gem dependency because we expect to be installed within the
	#Vagrant environment itself using `vagrant plugin`.
	gem "vagrant", '2.1.2',
	    :git => 'https://github.com/mitchellh/vagrant.git',
	    :ref => 'v2.1.2'

	gem 'rake','12.0.0'
end
