# Adding C dependencies to Zig

The other day I wanted to use sqlite and curl in a programming I'm working on in Zig.
I spent a lot of time looking up resources when it is extremely simple in practice.

## Needed Background

### Artifacts

The method `*std.Build.installArtifact` adds a dependency to install. This can be a library or a executable. The output can be found in the `zig-out` prefix.

> [!NOTE]
> The prefix for where artifacts are installed can be changed by passing `-p <path>` to `zig-out`

### Executables and libraries

In a `build.zig`, you can create a library, executable, or both. In his post, we will be adding C dependencies directly to our binary executable.

```build.zig
//...
    const exe = b.addExecutable(.{
        .name = "my_project_name",
        .root_module = exe_mod,
    });
    //if your c library depends on libc, link that as well.
    exe.linkLibC();
//...
```

## Considerations

When adding a C library, you have two options, link the system version library or import the existing C source code. If it is easy to compile from source, like SQLite, I suggest compiling it from source as you then have full control over what version of the library your program uses. If a program is widely used and comes packaged with your target operating systems, link the system version. Sometimes, you will forced to compile from source. This will require a lot more tinkering and reverse engineering make files.

## Linking with system libs

Mostly all UNIX systems come with Curl, so linking it via the system is very safe.
Always consider which systems you are targeting.

```build.zig
//Zig searches /usr/include for c packages on linux
exe.linkSystemLibrary("curl");
```

```main.zig
const c = @cImport(@cInclude("curl/curl.h"));
const std = @import("std");

pub fn main() !void {
    if (c.curl_global_init(c.CURL_GLOBAL_DEFAULT) != 0) {
        std.debug.print("curl_global_init failed\n", .{});
        return;
    }

    const curl = c.curl_easy_init();
    if (curl == null) {
        std.debug.print("curl_easy_init failed\n", .{});
        c.curl_global_cleanup();
        return;
    }

    // Set URL
    _ = c.curl_easy_setopt(curl, c.CURLOPT_URL, "https://www.google.com");

    // Follow redirects (optional)
    _ = c.curl_easy_setopt(curl, c.CURLOPT_FOLLOWLOCATION, .{1});

    // Perform request
    const res = c.curl_easy_perform(curl);
    if (res != c.CURLE_OK) {
        std.debug.print("curl_easy_perform() failed: {s}\n", .{c.curl_easy_strerror(res)});
    }

    // Clean up
    c.curl_easy_cleanup(curl);
    c.curl_global_cleanup();
}

```

Very intuitive

## Linking from source

In this example, I use [SQLite](https://sqlite.org/download.html). SQLite comes has a amalgamation version, which is just all the SQLite source code in one giant C file.
This is very convenient for embedding in our project.

```build.zig
  //if you have multiple C source files, use `addCSourceFiles` instead
  exe.addCSourceFile(.{ .file = b.path("src/sqlite_v3.50.4/sqlite3.c") });
  exe.addIncludePath(b.path("src/sqlite_v3.50.4"));
  //Dont't forget to add the header fies
  exe.installHeader(b.path("src/sqlite_v3.50.4/sqlite3.h"), "sqlite3.h");
```

```main.zig
const std = @import("std");
const lib = @import("sqlite_lib");
const sqlite = @cImport(@cInclude("sqlite3.h"));

const SETUP =
    \\CREATE TABLE Bugs
    \\(
    \\ name TEXT,
    \\ type TEXT
    \\);
;

pub fn main() !void {
    var conn: ?*sqlite.sqlite3 = undefined;
    var rc = sqlite.sqlite3_open("data.db", &conn);
    if (rc != sqlite.SQLITE_OK) {
        const err = sqlite.sqlite3_errmsg(conn);
        std.debug.print("{s}\n", .{err});
    }
    defer _ = sqlite.sqlite3_close(conn);

    var err: [*c]u8 = undefined;
    rc = sqlite.sqlite3_exec(conn, SETUP, null, null, &err);
    if (err != null or rc != sqlite.SQLITE_ABORT) {
        std.debug.print("{s}\n", .{err});
        sqlite.sqlite3_free(err);
    }
}
```

## Conclusion

Adding C dependencies to Zig is very easy and intuitive.
Now you try to make something cool.

Checkout the [Zig build system tutorial](ziglang.org/learn/build-system) for more about the build system.
