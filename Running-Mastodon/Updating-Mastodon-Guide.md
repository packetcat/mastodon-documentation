# Updating Mastodon

This guide assumes you have the set up seen in the [Production Guide](./Production-Guide.md).

## Read the release notes

Please read the [release notes](https://github.com/tootsuite/mastodon/releases/) first.

The release notes may contain special instructions that you may need to run for any given
new Mastodon release.

## Update the git repository
After you are done reading the release notes, you will want to log in as the mastodon user
and update the git repository.

This is how you do that:

```sh
su - mastodon
cd ~/live
git pull
git checkout $(git tag -l | sort -V | tail -n 1)
```

The above commands will update your copy of your git repository to the latest available
tagged release.

Always run tagged releases on production Mastodon instances. Never run the master branch
on your production instance.

You can always use `git status` to verify what tagged release your local 
repository copy is at, for example:

```sh
mastodon@mastodon:~/live$ git status
HEAD detached at v1.4.7
nothing to commit, working directory clean
```

The above output indicates that my local repository copy is using the tagged release 'v1.4.7'
of Mastodon.

## Ruby and node.js dependency updates

You may need to update Ruby and node.js upon upgrading to a new Mastodon release.
The aforementioned release notes will mention if you need to do the rest of this section.

This is how you update Ruby dependencies:
```sh
cd ~/live
bundle install
```

This is how you update node.js dependencies:
```sh
cd ~/live
yarn install
```

## Database schema updates

You may need to update database schema upon upgrading to a new Mastodon release.
The aforementioned release notes will mention if you need to do the rest of this section.

This is how you run a database migration:
```sh
cd ~/live
RAILS_ENV=production bundle exec rails db:migrate
```
