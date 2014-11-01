# Salt
A fork of [saltstack/salt](https://github.com/saltstack/salt).
  
## Contents
- [Add state id prefix](#add-state-id-prefix)
- [Add relative path syntax for salt scheme](#add-relative-path-syntax-for-salt-scheme)

## Add state id prefix
Add **env:sls::** prefix to each state id so that same state id value in different sls files would not conflict.

For example, files **a.sls** and **b.sls** both have defined a state for installing **sudo**.
```
sudo:
  pkg.installed
```
This will cause the error of conflicting ids.
```
Detected conflicting ids, SLS ids need to be globally unique.
The conflicting id is 'sudo' and is found in SLS 'base:a' and SLS 'base:b'
```

Each time adding a state id would risk conflicting with existing ones.  
Given the idiomatic use of state id as default value for argument **name**, conflicting ids are not uncommon.  
To avoid the confliction, we have to rename one of the state ids, and supply **name** argument explicitly.
```
sudo2:
  pkg.installed
    - name: sudo
```
This is neither convenient nor elegant. More importantly, modularity hurts.

I suggest the salt program should automatically prefix each state id with their **env** and **sls** values, For example, the env value is **base**, and the sls value is **a** and **b** for files **a.sls** and **b.sls** respectively, then the **sudo** state id in the two files become ```base:a::sudo``` and ```base:b::sudo```. This way state ids defined in different sls files would never be conflicting with each other.

### How to implement
The [change](https://github.com/AoiKuiyuyou/salt/commit/1d866cbb9e258f949f12f82578dee25745bf752b) is added in branch [aoik_stateidprefix](https://github.com/AoiKuiyuyou/salt/tree/aoik_stateidprefix).

In the function [salt.state::merge_included_states](https://github.com/AoiKuiyuyou/salt/blob/1d866cbb9e258f949f12f82578dee25745bf752b/salt/state.py#L2767), given a state dict like this
```
{
   'sudo': {
       '__env__': 'base',
       '__sls__': 'a',
       'pkg': [
           'installed',
           {
               'order': 10000,
           }
       ]
   }
}
```
we will prefix its state id with **env** and **sls** values like this
```
{
   'base:a::sudo': {
       '__env__': 'base',
       '__sls__': 'a',
       'pkg': [
           'installed',
           {'order': 10000},
           {'name': 'sudo'},
       ]
   }
}
```
Also notice the no-prefix state id, i.e. **sudo**, is used as the default value for argument **name** if **name** is not explicitly specified.

## Add relative path syntax for salt scheme
Notice with the introdution of variable **slspath**, this change is no longer needed.  
I just put it up for idea sharing purpose.

More info about variable **slspath**:
- [https://github.com/saltstack/salt/issues/815](https://github.com/saltstack/salt/issues/815)
- [https://github.com/saltstack/salt/pull/14535](https://github.com/saltstack/salt/pull/14535)
- [https://github.com/saltstack/salt/issues/14809](https://github.com/saltstack/salt/issues/14809)


The [change](https://github.com/AoiKuiyuyou/salt/commit/c26f81bec7786be385cc88902bb95b46d5648ba2) is added in branch [aoik_relpath](https://github.com/AoiKuiyuyou/salt/tree/aoik_relpath).

The idea is we use ```salt://.``` for relative path, for exmaple ```salt://./doc.txt```.  
Notice the dot ```.``` after the normal ```salt://``` scheme prefix. When there is a dot, we consider it a relative path, and convert it to the corresponding absolute form.

### How to implement
In the function [salt.state::merge_included_states](https://github.com/AoiKuiyuyou/salt/blob/c26f81bec7786be385cc88902bb95b46d5648ba2/salt/state.py#L2767), if argument **source** is present in relative form, convert it to absolute form, according to the state's **sls** value.