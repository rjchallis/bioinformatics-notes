---
scope:
  - rjchallis/bioinformatics-notes
tags:
  - git
---

# Synchronise notes across repositories

I like to keep notes in a collection of markdown files, some of these files are useful in different contexts so I use markdown metadata and git hooks to help keep notes synchronised across multiple repositories. For this to work, I only add/edit notes to one source repository and treat the notes directory in the destination repositories as immutable.

I keep local copies of each repository organised by organisation, e.g.:

```
projects
  - all-notes
  - organisation1
    - project1
    - project2
  - organisation2
    - project3
    - project4
  - rjchallis
    - bioinformatics-notes
```

Given a source repo `all-notes` with a structure like:

```
all-notes
  - general
    - notesA.md
    - notesB.md
  - project1
    - notesC.md
    - notesD.md
  - project2
    - notesE.md
    - notesF.md
  ...
```

Notes are flagged for sharing with other repositories by listing the other repositories under the `scope` heading in the file metadata:

`all-notes/general/notesA.md`

```markdown
---
scope:
  - organisation1/project1
  - organisation2/project3
---

# Example notes

These notes are flagged for sharing with `organisation1/project1` and `organisation2/project3`
```

The post-commit hook script automates the process of copying, commiting and pushing the notes to the additional repositories. The notes are copied to a `notes/<github_username>` directory in the destination repositories unless the `github_username` of the destination repository matches the `github_username` of the `all-notes` repository and the repository name contains `notes`. For example these notes are copied directly to the root directory of `rjchallis/bioinformatics-notes`.

## Usage

To set up the commit hook, create a post-commit hook script in a source notes repository and make it executable. This will run after each commit synchronise any markdown files that were tagged in that commit with destination repositories listed in the metadata `scope`.

```
cd all-notes
.git/hooks/post-commit
chmod 755 .git/hooks/post-commit
nano .git/hooks/post-commit
```

.git/hooks/post-commit

```bash
#!/bin/bash

# Get git username from remote url
GIT_NAME=$(git remote get-url origin | cut -d'/' -f 4)

# activate conda enve with yq command
eval "$(conda shell.bash hook)"
conda activate notes_env

# List markdown files changed in the last commit
FILES=$(git diff-tree --no-commit-id --name-only -r HEAD | grep "..*/..*\.md$")

# Loop through modified files
for FILE in $FILES; do
  # Read scope from metadata
  while read -r DIR; do
    if [ ! -z "$DIR" ]; then
      # Copy modified file to project repository, commit and push
      BASENAME=$(basename $FILE)
      if [[ $DIR == *"$GIT_NAME"*"notes"* ]]; then
        DEST=$BASENAME
      else
        DEST=notes/$GIT_NAME/$BASENAME
      fi
      echo copying $FILE to $DIR
      rsync $FILE ../$DIR/$DEST
      git -C ../$DIR/ pull &&
      git -C ../$DIR/ add $DEST &&
      git -C ../$DIR/ commit -m 'add notes' &&
      git -C ../$DIR/ push
    fi
  done < <(awk -v n=2 '/^---$/{n--}; n > 0' $FILE | tail -n+1 | yq -r '.scope[]') 2> /dev/null

done
```
