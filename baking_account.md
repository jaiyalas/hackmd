# Baking Account 

author: Chiachi, YunYan, GA
date: 2021/03/18

## tl;dr

This draft states only the very first step of Baking Account. Further Baking Account features can be built upon this. The goal is to allow users to register his/her public keys as consensus key and spending key. The idea is as follows:

1. adding a key-value map to storage, from key hash to a pair of consensus key and spending key
2. adding new operation for manipulating(CRUD)
3. differential right of keys. The consensus key is for all kind of operations while the spending key is for transaction operation only.


## New operation (CRUD with storage)

Notations:

- `K` : represent a public key in Tezos
- `KH` : represent a key hash, i.e. tz{1,2,...} account
- `Kc` : represent consensus key
- `Ks` : represent spending key
- `a|b` : means `a` or `b`
- `{key, value}` : a key-value record

### Create

- `f` : function computing `KH` for a given `K`

```
f(K) -> KH
```

- `add` : given `Ks`, `Kc` and `KHc = f(Kc)`, if `KHc` doesn't exist in storage, `add` will create a key-value record, `{KHc, (Kc, Ks)}`, into storage

```
add(Kc, Ks) ->
  let KHc = f(Kc)
  if (KHc exists in storage)
    reject
  else
    // save record
    {KHc, (Kc, Ks)} 
```

~~- **reversed mapping**: we'd also save a  record in storage that from a *key* to all of baking accounts it can manage or spend.~~

~~- Question : should we have *reversed mapping*? If not, it means users need to remember their key hash.~~

### Update

```
update (Kc'|Ks', Kc, KH) ->
  if Kc is the consensus key of KH
    //update KH with Kc'|Ks' in stoarge
    {KH, Kc'|Ks'} 
  else
    reject
```

~~- Question: same question as above, *reversed mapping*?~~

### Delete (maybe no need ?)

```
delete (KH, Kc) ->
  delete {KH, (Kc, Ks)} from stoarge
```

~~- Question: same question as above, *reversed mapping*?~~

### Read (maybe no need ?)

```
print_KH(KHi) -> { KHi, (Kci, Ksi) }
```

~~- Question: same question as above, *reversed mapping*?~~


## Perform operation

```
perform_operation(KH, Kc|Ks, op) ->
  case
    KH maps no keys && check manager signature -> 
      exec operation (as current behavior)
    KH maps to kc|ks && kc|ks has permission && check manager signature with kc|ks -> 
      exec operation
    otherwise -> reject
```