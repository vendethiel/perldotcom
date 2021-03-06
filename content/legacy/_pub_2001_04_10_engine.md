{
   "thumbnail" : null,
   "tags" : [
      "array",
      "perl",
      "search"
   ],
   "date" : "2001-04-10T00:00:00-08:00",
   "image" : null,
   "title" : "Designing a Search Engine",
   "categories" : "data",
   "slug" : "/pub/2001/04/10/engine.html",
   "description" : " A couple of months ago, I was approached by a company that I had done some work for previously, and was asked to build them a search engine to index about 200MB of HTML. However, some fairly strict rules...",
   "draft" : null,
   "authors" : [
      "pete-sergeant"
   ]
}



A couple of months ago, I was approached by a company that I had done some work for previously, and was asked to build them a search engine to index about 200MB of HTML. However, some fairly strict rules were laid down about what I could and couldn't do. First, the search engine had to be built from scratch, so that the rights to it were fully theirs. Second, I wasn't allowed to use any non-standard modules. Third, the data had to be held in plain-text files. Fourth, I was assured that all DB\* modules were not working, and couldn't be made to work, which seemed somewhat surprising. And finally, one had to be able to perform Boolean searches, including the use of brackets.

As you can imagine, this presented quite a challenge, so I accepted and got to work. I want to discuss the two most interesting parts of the project: how to store and retrieve the data efficiently, and how to parse search terms.

Because of the ideas I had on parsing the search terms, which will be discussed later, I needed a system that could take a word and send back a list of all the files it appeared in and how many times it appeared in them. This had to be as fast as possible without using insane amounts of disk space. Second, I needed to be able to get information about the files indexed quickly without actually opening them - the files could be large, and could be held remotely.

The first thing to do was to assign each indexed file a unique number - this would allow me to save space when referring to it, and would allow me to easily look up information on the file. Each word would have a list of these numbers listed under it, so I could take the word 'camel' for example, and get a list of file numbers back.

To me, the most obvious way of implementing this was to use a tied hash. However, my project rules stated I wasn't allowed to do that. I tried to think of other ways to implement a tied-hash system without using any of the DB or DBI modules. My first idea was to piggy back off a similar feature used in almost all operating systems; or, to be more precise, every file system. By creating each word as a file I could easily `open (DATA, "camel")` to get my data.

At this point, I had two types of indexes: one that listed summary information for each file by file number, and another that held information about each word.

There were two problems, however, with using the file system as my hash. While with a small number of words it was quite fast, most operating systems use a linear search to locate files in a directory, so opening "zulu" in a 10,000 file directory quickly became quite slow. The other problem was minimum file sizes, especially under Windows. This is a huge problem when your minimum file size is 16kb - as it is on fat16 with a 1GB hard drive - as 100 files translates as 1.6MB. When the data you're indexing is 20MB and you get about four times this much worth of index files, you're doing something wrong.

The solution for the file numbers was quite easy: I would spit out the file offsets of where information about each file was stored in my file index. Then, I'd read these offsets in at the start of each search, so that if I wanted information about file 15, I could get the offset by saying $file\[15\] and then seeking to that point.

I was beginning to despair for an elegant solution to the word indexing until Simon Cozens pointed out the wonderful Search::Dict. Search::Dict is a standard module that searches for keys in dictionary files. Essentially, it allows you to give it a word and a file-handle, where the file-handle is tied to a bunch of lines in alphabetical order, and will then change the next read position of the file-handle to the line that starts with the word. For example:

    look *INDEX, "camel";
    $line = <INDEX>;

would assign $line to the first line in the file handle beginning with camel. In addition, since it uses an effective binary search to achieve this, retrieval of data was fast. Problem solved.

Parsing of the Boolean search terms was the most difficult part of the program. Users had to be able to enter terms such as cat and (dog not sheep) and get sane results. The only way I could deal with this, I decided, was to go along the search terms from left to right and keep an array of file numbers that were still applicable.

To do this, I created three subroutines that I called `&array_and`, `&array_not`, and `&array_both`. `&array_and` would take our current results array and add the list of file numbers from a given word. `&array_not` would 'subtract' the file numbers from one word from our results array, and `&array_both` would return shared elements between the results array and the file numbers from the search word.

I ripped a lot of code from the Perl Cookbook to make these array functions. The code for these functions can be seen below:

    sub array_both {
     my $prev = 'nonesuch';
     my @total_first = (@{$_[0]}, @{$_[1]});
     @total_first = sort(@total_first);
     my $current;
     my @return_array;
     foreach $current (@total_first) {
      if ($prev eq $current) { push(@return_array, $prev);  }
      $prev = $current;
     }
     @return_array = @{&add_array(\@return_array, \@return_array)};
     return \@return_array;
    }

    sub array_and {
     my $prev = 'nonesuch';
     my @total_first = (@{$_[0]}, @{$_[1]});
     my %seen = (); # These next two lines are from the PCB
     my @return_array = grep { ! $seen{$_} ++ } @total_first ;
     return \@return_array;
    }

    sub array_not {

     # Again this is ripped straight from PCB

     my @a = @{$_[0]};
     my @b = @{$_[1]};
     my %hash = map {$_ => 1} @a;
     my $current;
     foreach $current (@b) {
      delete $hash{$current};
     }
     @a = keys %hash;
     @a = sort(@a);
     return \@a;
    }

The only big problem left was dealing with brackets. The only solution I could come up with was to make the search term parsing code a subroutine that returned an array. This way when I came to some logic in brackets, I could send it back to the subroutine, from which I would get a list of words, exactly like if the logic had been a single word. The main problem that I envisaged with this would be getting Perl to deal with expressions such as `sheep and (dog not cat) and (camel not panther)`. How did I get Perl to match just the first set of brackets, or, if nested brackets were present, to match all the logic? Damian Conway has written an excellent module called Text::Balanced, which I was just about to start using, before the project specifications changed(!) and I was told we no longer needed to allow nested-bracket searching.

Yet again, Perl came into its own when writing the search engine. The availability of solutions to my problems in the form of modules saved me a lot of time, and saved the task from being inundated with my own, rather bad, code. The ability to use Perl to quickly extract titles from HTML documents and strip HTML tags in very few lines of code also made my life far easier.
