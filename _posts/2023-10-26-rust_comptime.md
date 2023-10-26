---
title: "Rust: Entity Proc Macro I - Compile-time Reflection"
date: 2023-10-26
categories: [Blogging, Programming]
tags: [Rust]
---

Time for an _actual_ blog post!

## Background Information

In my [Discord bot](https://github.com/TeamHarTex/HarTex), I have to cache certain objects sent from Discord included in their
gateway payloads. While I use the amazing [Twilight](https://github.com/twilight-rs/twilight) library, their "Tier 1 crate" for
in-memory caching didn't suffice for me. For one, as inferred from the name, the objects are cached in RAM, which is volatile and
not persistent. Things like guilds or the current user may exist as its volatile form - as all of these do not have the requirement
to be cached persistently; but when messages come in to play, the situation is different.

A planned feature for my bot is the ability to delete messages that may be, perhaps days old, weeks old or months old. If the messages
were cached in a volatile way, all the information would be lost after a bot reboot and is definitely no good for this feature to work.
Indeed, using the Discord REST API isn't all that good either: there is a limit to how much messages can one fetch, and it is required
to specify certain time markers if the messages in question are older.

Additionally, the in-memory cache library forces me to cache _everything_, which uses more memory than it should be. Therefore, I opted
to writing my own cached objects and implemented a PostgreSQL-based persistent caching system.

## Problem: The Entities

Some implementations of entities are as follows, where `Entity` is a derive macro that implements the `Entity` trait:

```rust
#[derive(Entity)]
pub struct MemberEntity {
    #[entity(id)]
    pub guild_id: Id<GuildMarker>,
    pub roles: Vec<Id<RoleMarker>>,
    #[entity(id)]
    pub user_id: Id<UserMarker>,
}

impl From<(Member, Id<GuildMarker>)> for MemberEntity {
    fn from((member, guild_id): (Member, Id<GuildMarker>)) -> Self {
        Self {
            guild_id,
            roles: member.roles,
            user_id: member.user.id,
        }
    }
}

#[derive(Entity)]
pub struct GuildEntity {
    pub default_message_notifications: DefaultMessageNotificationLevel,
    pub features: Vec<GuildFeature>,
    pub icon: Option<ImageHash>,
    #[entity(id)]
    pub id: Id<GuildMarker>,
    pub large: bool,
    pub name: String,
    pub owner_id: Id<UserMarker>,
}

impl From<Guild> for GuildEntity {
    fn from(guild: Guild) -> Self {
        Self {
            default_message_notifications: guild.default_message_notifications,
            features: guild.features,
            icon: guild.icon,
            id: guild.id,
            large: guild.large,
            name: guild.name,
            owner_id: guild.owner_id,
        }
    }
}
```

You may notice that the code is too repetitive, and I am a proverbially "lazy" person... so I wrote a procedural macro
to generate the boilerplate for me.

## Plan: The Procedural Macro

```rust
#[entity(
    from = "",
    id = [],
    exclude = [],
    include = [],
    extra = [],
    overrides = [],
)]
```

- The `from` field specifies the `twilight-model` type to generate the fields from, as well as the `From` implementation.
- The `id` field specifies the fields that form the unique ID for this entity. This will be used for the `Entity::Id` associated
type.
- The `exclude` field specifies the fields **NOT** to include in the entity.
- The `include` field specifies the fields to **BE** included in the entity. Note that this and the `exclude` fields are mutually
exclusive.
- The `extra` field specifies extra fields to be included.
- The `overrides` field specifies the types to override import paths for if included. This is particularly useful if the generated
path without overriding contains a private module.

## Step 1: Generating Type Metadata for Compile-time Reflection

### Step 1a: Downloading the Crate to Generate Metadata For

For compile-time reflection to be feasible in the first place, there have to be a way to obtain type information.

The type information is generated from a downloaded version of the `twilight-model` crate, of which version can be specified via a
feature flag, like `discord_model_v_0_15_4` for example:
```rust
#[cfg(feature = "discord_model_v_0_15_4")]
const MODEL_CRATE_VERSION: &str = "0.15.4";

reqwest::blocking::get(format!("https://github.com/twilight-rs/twilight/archive/refs/tags/twilight-model-{MODEL_CRATE_VERSION}.zip")).expect(&format!("twilight-model {MODEL_CRATE_VERSION} is not found"));
```

The downloaded archive is then extracted into the `downloaded` folder of the crate root:
```rust
let output_dir = Path::new("downloaded");
extract_archive(reader, output_dir);
```

The `extract_archive` function just opens it and extracts it file by file, to the `downloaded` folder as aforementioned:
```rust
fn extract_archive<R: Read + Seek>(reader: R, output_dir: &Path) {
    let mut archive = ZipArchive::new(reader).expect("failed to open zip archive");

    for i in 0..archive.len() {
        let mut file = archive
            .by_index(i)
            .expect("failed to obtain file in zip archive");

        if !file.name().starts_with(&format!(
            "twilight-twilight-model-{MODEL_CRATE_VERSION}/twilight-model"
        )) {
            continue;
        }

        let file_path = output_dir.join(file.name());

        if file.name().ends_with('/') {
            fs::create_dir_all(&file_path).expect("failed to create directory");
        } else {
            let mut output_file = File::create(&file_path).expect("failed to create file");
            io::copy(&mut file, &mut output_file).expect("failed to copy file from zip");
        }
    }
}
```

### Step 1b: Building Module Tree

The next step is to obtain the module structure of the crate. This is done by traversing the filesystem tree itself, starting from `lib.rs`.
```rust
let crate_dir = output_dir.join(format!(
    "twilight-twilight-model-{MODEL_CRATE_VERSION}/twilight-model"
));
let lib_rs_path = crate_dir.join("src/lib.rs");

let module_tree = build_module_tree_from_file(
    &lib_rs_path,
    &Visibility::Public(Token![pub](Span::call_site())),
);
```

The module tree is built using quite an amount of recursion, which I will skip here for now. For the full source code, you may have a look
at the corresponding file in the [GitHub repository](https://github.com/TeamHarTex/HarTex/blob/nightly/discord-frontend/hartex-discord-entitycache-macros/build.rs).

### Step 1c: Traverse the Module Tree and Generate Metadata Modules

This step traverses the module tree that was built in the previous step and generates a metadata module, containing type information of every single
`struct` and `enum` present, regardless of their visibility (spoiler alert: this is a problem).

As you can see, this _also_ uses recursion... I guess recursion is a common theme in this case. ._.
```rust
fn generate_metadata_from_module_tree(tree: &ModuleTree, nest: bool) -> TokenStream {
    let name = &tree.name;
    let items = &tree.items;
    let children = tree
        .children
        .iter()
        .map(|child| generate_metadata_from_module_tree(child, true))
        .collect::<Vec<_>>();

    let structs = items
        .iter()
        .filter_map(|item| ...)
        .map(generate_lazy_static_from_item_struct)
        .collect::<Vec<_>>();
    let enums = items
        .iter()
        .filter_map(|item| ...)
        .map(generate_lazy_static_from_item_enum)
        .collect::<Vec<_>>();

    if nest {
        quote! {
            #[allow(clippy::module_name_repetitions)]
            pub mod #name {
                use lazy_static::lazy_static;

                lazy_static! {
                    #(#structs)*
                    #(#enums)*
                }

                #(#children)*
            }
        }
    } else {
        quote! {
            use lazy_static::lazy_static;

            lazy_static! {
                #(#structs)*
                #(#enums)*
            }

            #(#children)*
        }
    }
}
```

Basically this part generates some of this code (not attaching all because that would be too long):
```rust
pub mod command { use lazy_static :: lazy_static ; lazy_static ! { pub static ref COMMAND : crate :: reflect :: Struct = crate :: reflect :: Struct { name : stringify ! (Command) . to_string () , generic_params : vec ! [] , fields : vec ! [crate :: reflect :: Field { name : stringify ! (application_id) . to_string () , vis : stringify ! (pub) . to_string () , ty : stringify ! (Option < Id < ApplicationMarker > >) . to_string () , } , crate :: reflect :: Field { name : stringify ! (default_member_permissions) . to_string () , vis : stringify ! (pub) . to_string () , ty : stringify ! (Option < Permissions >) . to_string () , } , crate :: reflect :: Field { name : stringify ! (dm_permission) . to_string () , vis : stringify ! (pub) . to_string () , ty : stringify ! (Option < bool >) . to_string () , } , crate :: reflect :: Field { name : stringify ! (description) . to_string () , vis : stringify ! (pub) . to_string () , ty : stringify ! (String) . to_string () , } , crate :: reflect :: Field { name : stringify ! (description_localizations) . to_string () , vis : stringify ! (pub) . to_string () , ty : stringify ! (Option < HashMap < String , String > >) . to_string () , } , crate :: reflect :: Field { name : stringify ! (guild_id) . to_string () , vis : stringify ! (pub) . to_string () , ty : stringify ! (Option < Id < GuildMarker > >) . to_string () , } , crate :: reflect :: Field { name : stringify ! (id) . to_string () , vis : stringify ! (pub) . to_string () , ty : stringify ! (Option < Id < CommandMarker > >) . to_string () , } , crate :: reflect :: Field { name : stringify ! (kind) . to_string () , vis : stringify ! (pub) . to_string () , ty : stringify ! (CommandType) . to_string () , } , crate :: reflect :: Field { name : stringify ! (name) . to_string () , vis : stringify ! (pub) . to_string () , ty : stringify ! (String) . to_string () , } , crate :: reflect :: Field { name : stringify ! (name_localizations) . to_string () , vis : stringify ! (pub) . to_string () , ty : stringify ! (Option < HashMap < String , String > >) . to_string () , } , crate :: reflect :: Field { name : stringify ! (nsfw) . to_string () , vis : stringify ! (pub) . to_string () , ty : stringify ! (Option < bool >) . to_string () , } , crate :: reflect :: Field { name : stringify ! (options) . to_string () , vis : stringify ! (pub) . to_string () , ty : stringify ! (Vec < CommandOption >) . to_string () , } , crate :: reflect :: Field { name : stringify ! (version) . to_string () , vis : stringify ! (pub) . to_string () , ty : stringify ! (Id < CommandVersionMarker >) . to_string () , } ,] } ; } }
```

Yeah, generated code that is... incredibly ugly. But hey, all of this code is generated by building an actual abstract syntax tree to convert from.

### Step 1d: Generate Struct and Enum Lookup Tables

Now that we have generated all the metadata, we need a lookup table such that we can find the metadata of a certain `struct` or `enum` just by peeking into the
lookup table.

```rust
fn generate_enum_metadata_map(tree: &ModuleTree) -> TokenStream {
    let paths = generate_module_path_from_tree("twilight_model", tree, ModuleTreeItemKind::Enum);
    let entries = paths
        .into_iter()
        .map(|path| ...)
        .collect::<Vec<_>>();
    quote! {
        fn create_enum_map() -> HashMap<&'static str, &'static crate::reflect::Enum> {
            let mut map = HashMap::new();
            #(#entries)*
            map
        }

        lazy_static::lazy_static! {
            pub(crate) static ref ENUM_MAP: HashMap<&'static str, &'static crate::reflect::Enum> = create_enum_map();
        }
    }
}

fn generate_struct_metadata_map(tree: &ModuleTree) -> TokenStream {
    let paths = generate_module_path_from_tree("twilight_model", tree, ModuleTreeItemKind::Struct);
    let entries = paths
        .into_iter()
        .map(|path| ...)
        .collect::<Vec<_>>();
    quote! {
        fn create_struct_map() -> HashMap<&'static str, &'static crate::reflect::Struct> {
            let mut map = HashMap::new();
            #(#entries)*
            map
        }

        lazy_static::lazy_static! {
            pub(crate) static ref STRUCT_MAP: HashMap<&'static str, &'static crate::reflect::Struct> = create_struct_map();
        }
    }
}
```

Just know that at the end, with everything combined - we now have a successfully generated type metadata file for use in the macro!

## Thoughts

It has been fascinating how the Rust compiler does not provide any compile-time reflection capabilities. To be fair, it would be pretty cool
to be able to inspect the type information during compilation (even from procedural macros, I think - as they are _indeed_ executed at _compile_ time).
[The Zig programming language](https://ziglang.org/) does provide some form of compile-time reflection via builtin functions, kind of like "compiler intrinsics"
in a sense - but it does allow programmers to do _powerful_ things.

I think that is about enough to wrap up today's post. Oh and also, this is the first post of a multi-post documentary of the abovementioned `entity`
macro. I originally wanted to include it in this post as well - but the type metadata generation already is long enough, so I don't want to bore you with more
Rust code dumped at you. XD

See you in "Entity Proc Macro II" - the title for which I have no concrete idea yet. That will probably come in early November, hopefully.

Hope you all enjoyed the "October 2023 Edition" of my blog posts! Feel free to leave comments down below if you have any questions or have any thoughts.