project('mir', 'c',
  version: '1.0.1',
  license: 'MIT',
  meson_version: '>=1.1',
  default_options: ['c_std=gnu11', 'warning_level=0'],
)

# Mirrors the GNUmakefile build of libmir.a, intended for consumption as a
# meson subproject wrap providing the 'mir' dependency.

cc = meson.get_compiler('c')

# MIR sources assume plain char is signed (matters on aarch64); the
# -fno-* flags are the GNUmakefile's gcc workarounds.
mir_c_args = cc.get_supported_arguments(
  '-fsigned-char',
  '-fno-tree-sra',
  '-fno-ipa-cp-clone',
  '-Wno-abi',
)

mir_deps = [
  cc.find_library('m', required: false),
  cc.find_library('dl', required: false),
]
if host_machine.system() == 'windows'
  mir_deps += [cc.find_library('kernel32'), cc.find_library('psapi')]
endif

mir_srcs = files('mir.c', 'mir-gen.c', 'mir-dwarf.c')
if get_option('c2mir')
  mir_srcs += files('c2mir/c2mir.c')
endif

libmir = static_library('mir', mir_srcs,
  c_args: mir_c_args,
  include_directories: '.',
  pic: true,
)

mir_dep = declare_dependency(
  link_with: libmir,
  include_directories: include_directories('.'),
  dependencies: mir_deps,
)
meson.override_dependency('mir', mir_dep)
