

# Conan Package Creation Basics

## Create conan package

Entry a directory of a recipe:

```shell
conan create . bakka/debug
```

Create a package using namespace of bakka/debug

## Add remote

Remote is a repository for storing the package

```shell
conan remote add bakka https://api.bintray.com/conan/davidzwb/bakka
```

Add a remote called bakka and specify add address as https://api.bintray.com/conan/davidzwb/bakka

Hit "SET ME UP" on the Bintray Conan repository to check the URL.

## Upload

Add credential of remote:

```shell
conan user -p [API Key] -r bakka davidzwb
```

API Key can be acquired on the "Edit Profile" page of Bintray

Upload the package to remote of bakka

```shell
conan upload cdcf/1.2@bakka/debug -r bakka
```



