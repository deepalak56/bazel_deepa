{% include "docs/repo/http-attrs.md" %}

<p>The <code>http_archive</code> rule downloads a file over HTTP, extracts it, and makes its contents available as a Bazel repository.</p>

<h4 id="http_archive_args">Arguments</h4>
<dl>
  <dt><a id="http_archive.name"></a><code>name</code></dt>
  <dd>
    <p>A unique name for this repository. All targets in this repository will be prefixed with
    <code>@&lt;name&gt;</code>.</p>
  </dd>
  <dt><a id="http_archive.urls"></a><code>urls</code></dt>
  <dd>
    <p>A list of URLs. Bazel tries to download the file from these URLs, in order, until it
    succeeds. The first successfully downloaded file is used.</p>
  </dd>
  <dt><a id="http_archive.sha256"></a><code>sha256</code></dt>
  <dd>
    <p>The SHA256 sum of the file. This must match the SHA256 sum of the file downloaded from the
    URL. If it doesn't, Bazel will emit an error and print the correct SHA256 sum.</p>
  </dd>
  <dt><a id="http_archive.type"></a><code>type</code></dt>
  <dd>
    <p>The archive type. Currently, Bazel supports <code>zip</code>, <code>jar</code>, <code>gzip</code>, <code>bzip2</code>, <code>xz</code>, <code>tar</code>, <code>tar.gz</code>, <code>tar.bz2</code>, and <code>tar.xz</code>. If not specified, Bazel guesses the type from the URL's file extension.</p>
  </dd>
  <dt><a id="http_archive.strip_prefix"></a><code>strip_prefix</code></dt>
  <dd><p>A non-empty directory name to strip from the extracted files. It is an error if the archive does not contain a directory with this name as its prefix. For example, if you download a tarball and it contains a top-level directory <code>foo-1.2.3/</code>, you should set this to <code>foo-1.2.3</code> to extract the contents directly, without the top-level directory. Only one of <code>strip_prefix</code> or <code>strip_components</code> can be set.</p></dd>
  <dt><a id="http_archive.strip_components"></a><code>strip_components</code></dt>
  <dd><p>An integer. Strip the given number of leading path segments from extracted files. For example, if set to <code>1</code>, the top-level directory will be removed. Only one of <code>strip_prefix</code> or <code>strip_components</code> can be set. Defaults to <code>0</code>.</p></dd>
  <dt><a id="http_archive.patches"></a><code>patches</code></dt>
  <dd>
    <p>A list of patch files that are applied to the extracted archive. The patches are applied in
    the order they appear in this list.</p>
  </dd>
  <dt><a id="http_archive.patch_args"></a><code>patch_args</code></dt>
  <dd>
    <p>A list of strings passed as arguments to the patch command. By default, <code>-p0</code> is passed.
    If you need to use <code>-p1</code>, you can set <code>patch_args = ["-p1"]</code>.</p>
  </dd>
  <dt><a id="http_archive.patch_cmds"></a><code>patch_cmds</code></dt>
  <dd>
    <p>A list of shell commands that are applied to the extracted archive after patches are
    applied. The commands are run in the order they appear in this list. These commands are run in
    the extracted directory, after stripping the prefix. You may use this to patch files that
    cannot be patched with the standard patch utility, for example to apply sed commands.</p>
  </dd>
  <dt><a id="http_archive.build_file"></a><code>build_file</code></dt>
  <dd>
    <p>The label of a file to use as the <code>BUILD</code> file for this repository. If not specified,
    Bazel looks for a <code>BUILD</code> file at the top level of the extracted archive.</p>
  </dd>
  <dt><a id="http_archive.build_file_content"></a><code>build_file_content</code></dt>
  <dd>
    <p>The content of a <code>BUILD</code> file to use for this repository. This can be used instead of
    <code>build_file</code> to specify the <code>BUILD</code> file content directly in the
    <code>WORKSPACE</code> file.</p>
  </dd>
  <dt><a id="http_archive.repo_mapping"></a><code>repo_mapping</code></dt>
  <dd>
    <p>A dictionary of repository name to its canonical name. For example, <code>{'@foo': '@bar'}</code>
    means that all occurrences of <code>@foo</code> in this repository will be replaced with
    <code>@bar</code>. This is useful for resolving conflicts between transitive dependencies.</p>
  </dd>
</dl>
