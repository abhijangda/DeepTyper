# DeepTyper

# Getting Started
## Data setup
A tab-separated list of repositories and the SHAs at the time of check-out (February 28) can be found in "repo-SHAs.txt". "cloner.sh" provides a bash script to clone all these repositories and reset to the SHA commit in this paper. Note that some repositories may have been deleted or made private, so these will not be cloned.
When all repositories have been checked out, please make a copy of the Repos directory named "Repos-cleaned". We will use this copy to store tokenized TypeScript data without type annotations, without altering the original clone.

Next, we create tokenized and type-aligned files in Repos-cleaned and extract these to train and test data. This step requires `node` and the `npm TypeScript` package (install using `npm install typescript`).
1. Run `node CleanRepos.js`. This will find any compilable solutions in the cloned projects (there may be more than one solution per project), type-check them and create corresponding tokenized data and type (`*.ttokens`) files in Repos-cleaned. It scrapes all user-added type annotations of the source code and stores these in `*.ttokens.pure` files.
2. Run `node GetTypes.js`. This will create three directories, each of which contains one file per solution, with each line corresponding to a TypeScript file in that solution. Each line starts with the space-separated TypeScript tokens of that file, followed by a tab, and then the space-separated types corresponding to those tokens (including "no-type" token "O").
   - `outputs-all` contains data in which every identifier is annotated with its inferred type. This will be used for training data.
   - `outputs-pure` contains only the real user-added type annotations for the TypeScript code (and `no-type` elsewhere); this is used for evaluation (GOLD data)
   - `outputs-checkjs` contains the TSc/CheckJS inferred types for every identifier. This can be used for comparing performance with CheckJS.

## Running DeepTyper
The code is implemented in [CNTK](https://github.com/Microsoft/CNTK).

First, run `lexer.py`. This will create a `data` directory with a random inter-project split of 80% train, 10% valid & 10% test data. It also creates a source (token) and target (type) vocabulary and a `test-projects.txt` file that can be used by some analysis scripts to retrieve which projects were chosen for testing purposes.
- By default, the lexer cuts of vocabularies at 10 tokens and drops files of more than 5K characters. To change these settings, please change those numbers on lines 9-11 of `lexer.py`

Secondly, to convert the train/valid/test data to CNTK compatible '.ctf' input files, please use:
``
python txt2ctf.py --map data/source_wl data/target_wl --input data/train.txt --output data/train.ctf
python txt2ctf.py --map data/source_wl data/target_wl --input data/valid.txt --output data/valid.ctf
python txt2ctf.py --map data/source_wl data/target_wl --input data/test.txt --output data/test.ctf
``

Finally, run `infer.py`, which will train the network over 10 epochs and print validation error at the end of every epoch. Note that the size of an epoch is currently set to ~17.9M train tokens, based on the data we used. If your `lexer.py` printed a different number for "Overall tokens: ... train", please change this number to ensure that the learning rate decay matches the real epoch length.

## Evaluation
Once the model is trained, choose the version with the best validation error and provide its path to `evaluation.py` on line 21. Running `evaluation.py` after this will simply load the corresponding model and iterate over all data in `outputs-gold` (with true, user-added annotations) and `outputs-checkjs` (to compare CheckJS' performance). It invokes the deep learner on the tokenized test data and compares the result with the true annotations (and CheckJS). It will write results to "results/evaluation.txt"; for each real type in the test data it writes the true type, CheckJS' type guess, DeepTyper's type guess, DeepTyper's confidence and DeepTyper's "rank". Most of our results can be computed from this table, including top-K accuracy and MRR (using the 'rank') and mix models' performance, hit and misses (using the 'confidence').

Two other evaluation scripts are included:
- `consistency.py` works the same as the above and prints DeepTyper's consistency for each file. Specifically, it shows how many tokens 1) are singletons in the file, so that they could not have multiple types; 2) have multiple occurrences in the file; 3) were inconsistently typed by DeepTyper, meaning at least two distinct types were assigned to the same name in the file. Note that we do not consider scoping, so some names assigned different types may actually really have different types (e.g. in different methods). These are fairly rare (~4.7% of all identifiers) and make these inconistency scores a slight over-approximation.
- `tokentypemodel.py` trains a name-only model to predict types using the MLE of types given that name in training data. Running this script prints top-K accuracies for K from 1 to 10 for this naive baseline.
