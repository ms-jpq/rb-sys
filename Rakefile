EXAMPLES = Dir["examples/*"]

def extra_args
  seperator_index = ARGV.index("--")
  seperator_index && ARGV[(seperator_index + 1)..-1]
end

namespace :test do
  desc "Run cargo test against current Ruby"
  task :cargo do
    begin
      cargo_args = extra_args || ["--workspace"]
      sh "cargo", "test", *cargo_args
    rescue
      if ENV["CI"]
        ENV["RB_SYS_DEBUG_BUILD"] = "1"
        sh "cargo", "test", *cargo_args
      end

      raise
    end
  end

  desc "Test against all installed Rubies"
  task :rubies do
    cmd = <<~SH
      gem install bundler:2.3.7 > /dev/null 2>&1
      bundle check || bundle install -j3 > /dev/null 2>&1
      bundle exec rake test:cargo
    SH

    sh "./script/xruby", "-c", cmd
  end

  namespace :examples do
    task :rust_reverse do
      cargo_args = extra_args || []
      envs = [{}, {"ALTERNATE_CONFIG_SCRIPT" => "extconf_bare.rb"}]

      Dir.chdir("examples/rust_reverse") do
        envs.each do |env|
          sh env, "rake", "clean", "compile", "test", *cargo_args
        end
      end
    end
  end

  desc "Test all examples against all installed Rubies"
  task examples: ["test:examples:rust_reverse"]

  desc "Run unit tests for the gem"
  task :gem do
    Dir.chdir("gem") do
      sh "rake"
    end
  end
end

desc "Run all tests"
task test: ["test:cargo", "test:gem", "test:examples"]

desc "Pretty the files"
task :fmt do
  sh "cargo fmt"
  sh "bundle exec standardrb --fix" if RUBY_VERSION >= "2.6.0"
  sh "npx prettier --write $(git ls-files '*.yml')"
  md_files = `git ls-files '*.md'`.split("\n").select { |f| File.exist?(f) }
  sh "npx", "prettier", "--write", "--print-width=120", "--prose-wrap=always", *md_files
end
task format: [:fmt]

desc "Lint"
task :lint do
  sh "bundle exec standardrb --format #{ENV.key?("CI") ? "github" : "progress"}" if RUBY_VERSION >= "2.6.0"
  sh "cargo fmt --check"
  sh "cargo clippy"
  sh "shellcheck $(git ls-files '*.sh')"
end

desc "Bump the gem version"
task :bump do
  require_relative "./gem/lib/rb_sys/version"
  old_version = RbSys::VERSION

  printf "What is the new version (current: #{old_version})?: "
  new_version = $stdin.gets.chomp

  sh "fastmod", "--extensions=toml", "version = \"#{old_version}\"", "version = #{new_version.inspect}"
  sh "fastmod", "--extensions=rb", "^  VERSION = \"#{old_version}\"", "  VERSION = #{new_version.inspect}"
  sh "cargo check"
  Dir.chdir("examples/rust_reverse") { sh("cargo", "check") }
  sh "bundle"
end

desc "Publish the crates and gems"
task :publish do
  Dir.chdir("gem") do
    sh "bundle exec rake release"
  end

  ["crates/rb-sys-build", "crates/rb-sys"].each do |dir|
    Dir.chdir(dir) do
      sh "cargo publish || true"
      sleep 30
    end
  end
end
