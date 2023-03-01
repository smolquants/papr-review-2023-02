# papr-review-2023-02

Economic review of [PAPR](https://papr.wtf).

See [`review/`](./review/) for the full review.


## Replication

To check the results, clone the repo

```sh
git clone https://github.com/smolquants/papr-review-2023-02.git
```

Install dependencies with [`hatch`](https://github.com/pypa/hatch) and [`ape`](https://github.com/ApeWorX/ape)

```sh
hatch build
hatch shell
(papr-review-2023-02) $ ape plugins install .
```

Setup your environment with an [Alchemy](https://www.alchemy.com) key

```sh
export WEB3_ALCHEMY_PROJECT_ID=<YOUR_PROJECT_ID>
```

Then launch [`ape-notebook`](https://github.com/ApeWorX/ape-notebook)

```sh
(papr-review-2023-02) $ ape notebook
```
