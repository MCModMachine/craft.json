craft.json
==========

`craft.json` is a specification for describing Minecraft mods, modpacks, etc.
`craft.json` is designed, first and foremost, to fit the large variety of
existing use cases in the modded Minecraft community. Note that while this is
the canonical representation, in practice this will just be used as a structure
instead of an actual JSON syntax in many situations. For example, pack devs
and modders will likely use a GUI that internally uses this format, and on the
server this data will likely be stored in an SQL database instead of in JSON
files. However, this spec still dictates the relevant data structures.

Project
=======

A project is the core unit of 

* `specVersion`: **number** always 1. Parsers must not attempt to parse files
  with other values for `specVersion`. This field is reserved for future use.
* `identifier`: **string** a unique alphanumeric (+ pipes) string representing
  the project. For mods, this should generally be the same as the Forge modid.
  <sup>1</sup> 
* `name`: **string** a human-readable name for the mod. This need not be unique.
* `description`: **string** a human-readable description of what a mod does.
* `website`: **string** a URL for a website about the mod.
* `type`: **string** the type of project. This may be:
  * `game`: the moddable game, presumably Minecraft. There is no reason this spec
    could not be repurposed for another game, however.<sup>4</sup>
  * `mod`: a individual mod. A `mod` should be exactly one JAR file.
  * `modpack`: a set of mods that can be played. Every instance has a 1-to-1
    mapping to a modpack. (Custom instances are just locally saved modpacks;
    customized modpacks are modpacks with a dependency on the base pack)
  * `modSet`: a set of seperate "mods" in the JAR/Forge sense that form one
    cohesive whole, such as Buildcraft or ProjectRed.
  * `resourcePack`: a resource pack.
* `version`: **string** the current version of the project. This string is
  purely for humans; there are no requirements about its format. Details of
  compatibility are expressed in the `compatibility` property. <sup>2</sup>
* `previousVersion`: **string** the version preceding this version of the
  project. Note that this should be based on what source code the current
  version was based off of, not release time. For example, version 5.3's
  `previousVersion` is 5.2, even if 5.3 is released after 6.0.
* `files`: **array of Files** the files that must be installed as part of this
  project, not including dependencies. Not required for `game` projects.
* `dependencies`: **array of Depencencies** the projects that must be installed
  for this project to function properly.
* `compatibility`: **object** this version of the project's compatibility with
  previous versions
  * `api`: **object** this version's API compatibility with the previous version
    * `upgrade`: **boolean** packages that expect previousVersion's API will
      work with this version. This should be false if API methods were removed.
    * `downgrade`: **boolean** packages that expect this version's API will work
      with previousVersion. This should be false if API methods were added.
  * `world`: this version's compatibility with world saves created with the same
    version.
    * `upgrade`: users can move from previousVersion to this version without a
      world reset (and items/blocks from this project will remain intact).
    * `downgrade`: users can downgrade from this version to previousVersion
      without a world reset (and items/blocks from this project will remain
      intact).
  * `balance`: balance changes between this version and the previous.
    * `noFeatures`: this update **does not** contain new features that could
      affect balance. This includes, but is not limited to, added blocks or
      items. <sup>3</sup>
    * `noRework`: this update **does not** contain a major rework or rebalance.
      <sup>3</sup>
* `doNotInclude`: **array of Uninclusions** list of projects that should not be
  included alongside this project.
* `license`: **object** a summary of the project's license. This should always
  reflect the actual legal license applied to the project.
  * `modpack`: **boolean/string** permissions granted by the license regarding
    inclusion in publically-released modpacks. May be `false` (do not allow), 
    `"permission"` (allow with prior permission), or `true` (allow).
* `side`: **string** the types of Minecraft instances where this project needs to
  be installed. May be `client`, `server`, or `both`. All project types default to
  `both`, except for `resourcePack` which defaults to `client`.

File
====

A File is a file or directory that needs to be added to an instance in the
process of installing a project.

* `method`: **string** how the file should be added to the instance. Additional
  properties are required depending on the method. Possible values are:
  * `copy`: simply copy the file into the instance.
    * `path`: **string** the path relative to the root of the instance,
      including filename, where the file should be installed.
  * `extract`: extract an archive to a certain location in the instance.
    * `path`: **string** the path relative to the root of the instance where the
      archive should be extracted.
    * `type`: **string** the format of the archive. No specific list of values
      are defined for this property, but the value should be the most common
      extension for the format, without the leading dot (for example, `zip` or
      `tar.gz`).
  * `forge`: only valid where the project's type is `mod`. Downloads the mod to
    a single local repository and tells Forge to include the mod via a command
    line argument. In cases where this is not supported, this is identical to
    `copy` with a path of `mods/{identifier}-{version}.jar`
  * `launchWrapper`: the file should be loaded into Minecraft with LaunchWrapper.
* `download`: **array of DownloadMethods** a list of methods by which the file
  may be downloaded. Note that this property may be set by the registry instead
  of in the original `craft.json` file provided by the modder. Clients should
  try each method in order before failing.
* `hash`: **string** the SHA1 hash of the file.

Download Method
===============

A description of how to download a file.

* `method`: **string** the method by which the file should be downloaded.
  Additional properties are required depending on the method. Possible values
  are:
  * `url`: the file is available by download via a URL.
    * `url`: **string** the URL from which the file may be downloaded.
  * `external`: the file must be supplied manually by the user. Modpack
    developers are advised not to use this feature to circumvent a modder's
    wishes.
    * `description`: **string** a human-readable string describing the file that
      must be provided.
    * `url`: **optional string** a URL from which the user may download the file.
    
Dependency
==========

A description of a project's requirements about the presence of another project.

* `project`: **string** the identifier of another project which this project
  depends on.
* `versions`: **array of strings** the versions of the other project with which
  this project is compatible. Note that you need not list every version which
  works; you only need to list multiple versions if you are compatible with, for
  example, multiple distinct versions of the API. If you are compatible with
  every verison of the mod that implements a certain version of the API, use
  `compatibility` to specify this. This allows your project to be compatible
  with newly-relesed versions of the project even if you don't explicitly say
  that it is, allowing players to get bugfixes and new features more quickly.
* `compatibility`: **array of strings** the requirements this project has about
  the included version of the other project. This array may include:
  * `api`: the included version must have the same API as the specified version.
  * `noFeatures`: the included version must have the same user-facing
    features as the specified version. Includes `noRework`.
  * `noRework`: the included version must not have had a major rebalance or
    rework since the specified version.
  * `bugfix`: the depending project requires a bugfix in the specifed version of
    the project; downgrading is not an option.
  * `exact`: the depending project requires exactly the version specified. Use
    with caution.
* `strength`: how strongly the depending project requires the depended project's
  existance. Possible values:
  * `required`: the depending project will break unless a compatible version of
    the depended project is available. Default for mods and resource packs.
  * `recommended`: the depended project should be installed by default, but if
    it is not installed due to an incompatibility or user preference then the
    depending project will still work. If the depended project is installed, it
    will always be of a compatibile version. Default for modpacks and mod sets.
  * `optional`: the depended package will not be installed by default, but if it
    is then it must be of a compatible version.
  * `integration`: only install the depending package if at least one depended
    package of type `integration` is installed.
    
Uninclusion
===========

An uninclusion is a declaration that one project should not be installed
alongside another.

* `project`: **string** the identifier of the project which should not be
  included.
* `type`: **string** may be `override` or `incompatible`. See below for
  explanations; other properties depend on the type.
  * A `override` uninclusion should be used when a mod explicitly replaces
    another mod, or in other situations where the project creator is confident
    that despite another project specifying a dependency as `required`, this
    project allows the instance to function without it. It simply causes the
    other projects' requirements for this project to be waived. Note that a
    client will never automatically use an overriding mod instead of the
    specified mod; a project must explicitly request that mod.
    * `versions`: **array of strings** the versions of the unincluded project
      that this project is capable of replacing. If another project needs a
      version of this project that is not able to be replaced, installation will
      fail. This may be ommited to represent all versions.
    * `compatibility`: **object** specifies this project's compatibility with
      projects that expect the other project's presence. This has the same
      syntax as Project's `compatibility` property.
  * A `incompatible` uninclusion should be used when two projects are
    incompatible. If two projects are incompatible, the client should try and
    find two versions which do work together, and failing that it should cause
    installation to fail.
    * `versions`: **array of strings** the versions of the
      unincluded project with which this project is incompatible. Versions which are
      not specified are assumed to still be usable. This may be ommited to represent
      all versions.
    * `warning`: **boolean** simply display a warning instead of preventing
      installation. Recommended for non-game-breaking bugs.
    * `message`: **string** custom message informing user of details of
      incompatibility.

Notes
=====

1. There are two cases in which an `identifier` would not be the same as the
   forge modid.
   1. New versions of mods should keep the same identifier, even if the modid
      is changed. For example, Applied Energistics 1 and 2 should be listed as
      the same mod with the same identifier, even though AE2 has a modid of
      appliedenergistics2.
   2. Because identifiers must be unique, forks of mods must use different
      identifiers. Defining who, if any, keeps the original name is a decision
      left to registries. The official registry's policy is here
      [link to be added  when we get to that point].
2. We don't use Semver for two reasons. First, some modders (such as Reika)
   prefer non-x.y.z versioning systems. Second, Semver has features designed for
   software libraries, not for Minecraft; Semver differentiates versions solely
   by API compatibility. However, mods and modpacks may have at least three
   different requirements on versions: API compatibility, world compatibility
   (the mod's ability to work in worlds created by previous versions of the mod)
   and balance changes (while kitchen sink packs probably don't need to avoid
   mod updates with new features, a pack like IE:E where a single item could
   throw off the balance of the pack would probably want a manual update by the
   pack team before mod updates with added items are rolled out).
3. Although these would be more simply phrased in the positive, they are phrased
   like this for consistency with the other flags where `true` indicates
   compatibility.
4. Your mod should only depend on the `minecraft` package if you are directly modifying
   the game (for example, if you are writing a non-Forge mod or a Forge mod that uses ASM).
   Other mods should simply depend on the relevant Forge version, so that their mods can still
   function if a new Minecraft version is released with no major changes.
   
