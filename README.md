# terraform-provider-artifactory repository upgrader

This is a simple script used to help with the migration of the old,
depreated repository types:

  - `artifactory_local_repository`
  - `artifactory_remote_repository`
  - `artifactory_virtual_repository`

to the type-specific replacements.  There's no great way to migrate a
repository from one type to another.  Therefore, we have to do surgery on
the terraform state file itself as well as the `*.tf` configs.

Please see the [DISCLAIMER](#DISCLAIMER) section below...

## How it works

This works by reading in a state file, finding all of the above resource
types, then renaming them all to their proper type-specific form based on
their settings.  It also finds dependency references from other items and
renames those.

This rename mapping is then used to also create new versions of all of the
`*.tf` files found in the current directory, renaming any references
found.  Since there's not a valid HCL library that can write HCL, this
works by just doing line-by-line screen-scraping of the code.  It's not
great, but works for us.  You might have to modify as appropriate.

## Caveats

This really only works if all of your repositories are defined in a single
terraform repo.  If they're managed via modules outside of the current
directory, it's not going to be able to handle that case.

## The procedure for using this ...

1. Make a copy of your state file locally.

    ```sh
    $ aws s3 cp s3://my-state-bucket/my-project/terraform.tfstate .
    ```

2. Remove your s3 backend config so terraform can look at your local state
   file instead.

    ```sh
    $ cat backend.tf
    terraform {
      backend "s3" {
        bucket         = "my-state-bucket"
        key            = "my-project/terraform.tfstate"
        region         = "us-west-2"
        dynamodb_table = "terraform-state-lock"
      }
    }
    $ mv backend.tf backend.tf-
    ```

3. Update your provider version to minimum required for your repository
   types:

   * nearly all repository types are supported in `2.25.0`
   * `artifactory_local_cargo_repository` added in `3.1.0`
   * `artifactory_local_conda_repository` added in `3.1.0`

4. Initialize your terraform repository so it is not using your real state
   store (just the safe backup that's local) and has the correct provider
   version:

    ```sh
    $ rm -rf .terraform
    $ terraform init
    $ terraform plan        # <== make sure this is clean!
    ```

5. Do a test conversion of all of your data first.  This will generate new
   `*.new` files locally with the updated versions so you can diff them
   and verify.

    ```sh
    $ cp ~/github/artifactory-repository-upgrade/convert-repo .
    $ ./convert-repo --state-file terraform.tfstate
    Converting terraform.tfstate...
    Converting repos1.tf...
    Converting repos2.tf...
    ...
    ```

6. Now compare the old and new files to see if things look right.  Maybe
   verify there are no references to the old types:

    ```sh
    $ for i in *.tf; do echo == $i ==; diff $i $i.new; done | less
    $ egrep 'artifactory_(local|remote|virtual)_repository' *.new
    ```

7. If things look good, you can take those `.new` files and just move
   them whereever you want or start over and let the tool replace the
   original files.

    ```sh
    $ rm *.new
    $ ./convert-repo --state-file terraform.tfstate --replace
    ```

8. Now the majority of the update should be done.  But the new resource
   types include different settings than the generic ones.  Refresh your
   local, temporary state file to get it in a good state:

    ```sh
    $ terraform apply -refresh-only
    ```

   You'll see a lot of things refreshing ... probably new settings that it
   didn't store before.  For us, it included settings like `xray_index`
   that terraform didn't even track before.

9. Now you can go through the tedious task of running a `plan` and updating
   the configs to make sure they are it only updates what you expect.

   We found the following new settings cause it to want to replace the
   entire repository:

   * `external_dependencies_patterns` on some reposotories (I don't
     understand why this wants to do a full replacement)

   The following are some of the other settings we had to manually add
   to our repository resource configs to keep them from getting updated.
   They were either new settings or their defaults changed.

   * `list_remote_folder_items`
   * `primary_keypair_ref`
   * `repo_layout_ref`
   * `retrieval_cache_period_seconds`
   * `tag_retention`
   * `vcs_git_provider` (e.g. "GITHUB" -> "ARTIFACTORY")
   * `xray_index`

10. Once you're done and your plan is clean (or only includes changes
    you'd expect), it's time to commit things for real.

    * make sure nobody else is touching the repository
    * apply your changes
    * copy your updated state file to the permanent location
    * restore your backend config
    * commit all changes to your repo

## DISCLAIMER

Note, this repository was used by myself to upgrade our configs.  It
requires manually modifying terraform state files and doing somewhat
advanced actions.  You should know exactly what you're doing in each step
before doing them.  You should maintain backups of all state files and
configs.  I am not liable for any loss of data or catastrophes that might
occur.
