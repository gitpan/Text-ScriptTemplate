NAME
     Text::ScriptTemplate - Lightweight processor for full-featured template

SYNOPSIS
     use Text::ScriptTemplate;

     $tmpl = new Text::ScriptTemplate;    # create processor object
     $tmpl->setq(TEXT => "hello, world"); # export data to template
     $tmpl->load($file);                  # loads template from named file
     $tmpl->pack(q{TEXT: <%= $TEXT; %>}); # loads template from in-memory data

     print $tmpl->fill;                   # prints "TEXT: hello, world"

     # load intermixed Perl script and text as a template
     $tmpl->pack(q{<% for (1..3) { %>i = <%= "$_\n"; %><% } %>});

     print $tmpl->fill;                   # prints "i = 1\ni = 2\ni = 3\n"

DESCRIPTION
    This is a varient of Text::SimpleTemplate, a module for template-based
    text generation.

    Template-based text generation is a way to separate program code and
    data, so non-programmer can control final result (like HTML) as desired
    without tweaking the program code itself. By doing so, jobs like website
    maintenance is much easier because you can leave program code unchanged
    even if page redesign was needed.

    The idea of this module is simple. Whenever a block of text surrounded
    by '<%' and '%>' (or any pair of delimiters you specify) is found, it
    will be taken as Perl expression, and will be handled specially by
    template processing engine. With this module, Perl script and text can
    be intermixed closely.

    Major goal of this library is to provide support of powerful PHP-style
    template with smaller resource. This is useful when PHP, HTML::Embperl,
    or Apache::ASP is overkill, but their template style is still desired.

INSTALLATION / REQUIREMENTS
    This module requires Carp.pm and FileHandle.pm. Since these are standard
    modules, all you need is perl itself.

    For installation, standard procedure of

        perl Makefile.PL
        make
        make test
        make install

    should work just fine.

TEMPLATE SYNTAX AND USAGE
    Any block of text surrounded by '<%' and '%>' will be handled specially
    by template processor.

    For block surrounded by '<%=' and '%>, it will be taken as simple perl
    expression, and will be replace by its evaluated result.

    For block surrounded by '<%' and '%>, it will be taken as part of
    control structure, and after all parts are merged into one big block, it
    will be evaluated and the result will be handled as output.

    Suppose you have a following template named "sample.tmpl":

        === Module Information ===
        <% if ($HAS->{Text::ScriptTemplate}) { %>
        Name: <%= $INFO->{Name}; %>
        Description: <%= $INFO->{Description}; %>
        Author: <%= $INFO->{Author}; %> <<%= $INFO->{Email}; %>>
        <% } else { %>
        Text::ScriptTemplate is not installed.
        <% } %>

    With the following code...

        use Safe;
        use Text::ScriptTemplate;

        $tmpl = new Text::ScriptTemplate;
        $tmpl->setq(INFO => {
            Name        => "Text::ScriptTemplate",
            Description => "Lightweight processor for full-featured template",
            Author      => "Taisuke Yamada",
            Email       => "tai\@imasy.or.jp",
        });
        $tmpl->setq(HAS => { Text::ScriptTemplate => 1 }); # installed
        $tmpl->load("sample.tmpl");

        print $tmpl->fill(PACKAGE => new Safe);

    ...you will get following result:

        === Module Information ===

        Name: Text::ScriptTemplate
        Description: Lightweight processor for full-featured template
        Author: Taisuke Yamada <tai@imasy.or.jp>

    If you change

        $tmpl->setq(HAS => { Text::ScriptTemplate => 1 }); # installed

    to

        $tmpl->setq(HAS => { Text::ScriptTemplate => 0 }); # not installed

    then you will get

        === Module Information ===

        Text::ScriptTemplate is not installed.

    You can embed any control strucure as long as intermixed text block is
    surround by set of braces. This means

        hello world<% if ($firsttime); %>

    must be written as

        <% do { %>hello world<% } if ($firsttime); %>

    If you want to know more on this, please read TEMPLATE INTERNAL section
    for the detail.

    Also, as you might have noticed, any scalar data can be exported to
    template namespace, even hash reference or code reference.

    Finally, although I used "Safe" module in example above, this is not a
    requirement. However, if you want to control power of the template
    editor over program logic, its use is strongly recommended (see the Safe
    manpage for more).

RESERVED NAMES
    Since template can be evaluated in separate namespace, this module does
    not have much restriction on variable or function name you define in
    theory.

    However, due to internal structure of this module, please consider all
    names starting with "_" (underscore) as reserved for internal usage.

METHODS
    Following methods are currently available.

    $tmpl = new Text::ScriptTemplate;
        Constructor. Returns newly created object.

        If this method was called through existing object, cloned object
        will be returned. This cloned instance inherits all properties
        except for internal buffer which holds template data. Cloning is
        useful for chained template processing.

    $tmpl->setq($name => $data, $name => $data, ...);
        Exports scalar data ($data) to template namespace, with $name as a
        scalar variable name to be used in template.

        You can repeat the pair to export multiple sets in one operation.

    $tmpl->load($file, %opts);
        Loads template file ($file) for later evaluation. File can be
        specified in either form of pathname or fileglob.

        This method accepts DELIM option, used to specify delimiter for
        parsing template. It is speficied by passing reference to array
        containing delimiter pair, just like below:

            $tmpl->load($file, DELIM => [qw(<? ?>)]);

        Returns object reference to itself.

    $tmpl->pack($data, %opts);
        Loads in-memory data ($data) for later evaluation. Except for this
        difference, works just like $tmpl->load.

    $text = $tmpl->fill(%opts);
        Returns evaluated result of template, which was preloaded by either
        $tmpl->pack or $tmpl->load method.

        This method accepts two options: PACKAGE and OHANDLE.

        PACKAGE option specifies the namespace where template evaluation
        takes place. You can either pass the name of the package, or the
        package object itself. So either of

            $tmpl->fill(PACKAGE => new Safe);
            $tmpl->fill(PACKAGE => new Some::Module);
            $tmpl->fill(PACKAGE => 'Some::Package');

        works. In case Safe module (or its subclass) was passed, its "reval"
        method will be used instead of built-in eval.

        OHANDLE option is for output selection. By default, this method
        returns the result of evaluation, but with OHANDLE option set, you
        can instead make it print to given handle. Either style of

            $tmpl->fill(OHANDLE => \*STDOUT);
            $tmpl->fill(OHANDLE => new FileHandle(...));

        is supported.

TEMPLATE INTERNAL
    Internally, template processor converts template into one big perl
    script, and then simply executes it. Conversion rule is fairly simple -
    If you have following template,

        <% if ($bool) { %>
        hello, <%= $name; %>
        <% } %>

    it will be converted into

        if ($bool) {
            $_handle->(q{
        hello, });
            $_handle->(do{ $name; });
            $_handle->(q{
        });
        }

    Note line breaks are preserved. After all conversion is done, it will be
    executed. And depending on existance of OHANDLE option, $_handle (this
    is a code reference to predefined function) will either print or buffer
    its argument.

NOTES / BUGS
    Nested template delimiter will cause this module to fail.

SEE ALSO
    the Safe manpage and the Text::SimpleTemplate manpage

CONTACT ADDRESS
    Please send any bug reports/comments to <tai@imasy.or.jp>.

AUTHORS / CONTRIBUTORS
     - Taisuke Yamada <tai@imasy.or.jp>

COPYRIGHT
    Copyright 2001. All rights reserved.

    This library is free software; you can redistribute it and/or modify it
    under the same terms as Perl itself.

