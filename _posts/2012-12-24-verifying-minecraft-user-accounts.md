---
layout: post
title: 'Verifing Minecraft User Accounts'
---
When I need to give my brain a rest, I like to play Minecraft on an interesting
server known as [Civcraft][1]. The unique thing about this server is that it is
an experiment in anarchy of sorts. There are no rules except not to exploit
software glitches that could give you an unfair advantage. Robbery, murder,
griefing and trolling of all sorts are completely legal within the rules of the
server. As a result, there have evolved complex and organic societies complete
with competing cities, marketplaces and even ad hoc police forces and bounty
hunters.

![Trapped in a Prison Pearl][7]

The server facilitates this with a few custom plugins like [PhysicalShop][2] for
buying and selling via item chests and [PrisonPearl][3], which serves as the
server supermax by letting you banish someone to [The End][4] and keep them as
a prisoner in an Ender Pearl.

As an amateur compared to the major players, I actually enjoy watching the
societal developments more than playing the game itself. One thing that is
interesting to me is the trading. Unless you play constantly and understand
what items people want and what kind of supply they are in, pricing can be
confusing. Especially since it's a barter system with no centralized currency.
It can also be difficult to find what you're looking for.

I needed little pet project to keep my skills sharp and decided to build a market
place app for the game: [Civtrade][5]. At first it was just a place to list
and search for shops, but I soon decided a bounty system would be useful.

Just listing shops didn't really need any sort of user accounts. It was all
anonymous.  Putting a bounty on someone was different. Users would need to be
able to verify that the person posting the bounty was legit, otherwise someone
could post as a prominent player, promising a huge reward for the bounty of
their enemy.

Mojang doesn't provide OAuth or any other means to link your users' accounts
with their in-game identities. This is a shame, and it's probably holding up a
few good ideas. Of course, it could be by design.

In any case, I needed to verify that the users of Civtrade are who they say they
are. One way to do that is to reverse engineer the Minecraft client and mimic it.
It turns out, that's pretty easy. There's a few URLs the client uses to POST
your login, password and client version number. A successful response includes
a timestamp, username, and a session id.

For my purposes, I just needed to ensure that they can get a successful
response and that the username matches the one they're signing up with.

```ruby
class MinecraftAccountVerifier
  require 'net/http'
  require 'uri'

  AUTH_URI = URI.parse('https://login.minecraft.net/').freeze
  CLIENT_VERSION = 13

  attr_reader :error

  # Public: verify an account as a true Minecraft account this user has access to.
  #
  # login    - the username or email used to log into the Minecraft client
  # username - both the username for this service and their in-game identity
  # password - password used to log into the Minecraft client
  def initialize(login, username, password)
    @login = login
    @username = username
    @password = password
  end

  def authentic?
    response = login.body

    if response =~ username_regex
      true
    else
      @error = login.body.chomp
      false
    end
  end

  private
  def username_regex
    Regexp.new(@username, 'i')
  end

  def request_parameters
    {
      'user' => @login,
      'password' => @password,
      'version' => CLIENT_VERSION
    }
  end

  def login
    request = Net::HTTP::Post.new(AUTH_URI.request_uri)
    request.set_form_data(request_parameters)

    http = Net::HTTP.new(AUTH_URI.host, AUTH_URI.port)
    http.use_ssl = true
    http.verify_mode = OpenSSL::SSL::VERIFY_PEER

    http.request(request)
  end
end
```

This has the obvious downside that you're asking your users for their credentials
to a third party. They should never have to just trust you. I posted what I had
on reddit and after about an hour with some interest but no signups I disabled
it completely for the time being, let people sign up without verifying and
planned to add a less invasive verification step as soon as I could think of
one.

Then I started thinking. At the very core of it, I just needed someone to prove
to me that they are in control of a Minecraft user whose name matches their
username on Civtrade. Maybe change their Minecraft account email to a generated
throwaway briefly?  No, that's just as bad or worse than getting their
password. How about in-game verification? That would take way too much effort
to meet and verify people individually. And then it hit me: Minecraft lets you
customize your character by uploading a skin. A skin is just a tiny,
66&times;34px png image. I can have my users temporarily upload a unique
verification skin to their profile. Then I just have to diff the image against
the original.

```ruby
class MinecraftAccountVerifier
  require 'net/http'
  require 'uri'

  SKINS_S3_BUCKET = 's3.amazonaws.com'.freeze

  attr_reader :error

  # Public: verify an account as a true Minecraft account this user has access to.
  #
  # username - both the username for this service and their in-game identity
  def initialize(username)
    @username = username
  end

  def authentic?
    if user_skin
      return true if skin_difference == 0.0

      @error = "Skin does not match verification skin. Please wait a minute or try uploading the skin again. (#{skin_difference}% different)"
      false
    else
      @error = 'Your skin was not found. Please note that your username is case sensitive'
      false
    end
  end

  private
  def skin_difference
    diffs = []
    user_skin.height.times do |y|
      user_skin.row(y).each_with_index do |pixel, x|
        diffs << [x, y] unless pixel == verification_skin[x, y]
      end
    end
    diffs.length.to_f / verification_skin.pixels.length * 100
  end

  def user_skin
    @user_skin ||=
      Net::HTTP.start(SKINS_S3_BUCKET) do |http|
        response = http.get("/MinecraftSkins/#{@username}.png")
        if response.code == '200'
          datastream = ChunkyPNG::Datastream.from_blob(response.body)
          ChunkyPNG::Image.from_datastream(datastream)
        end
      end
  end

  def verification_skin
    @verification_skin ||=
      ChunkyPNG::Image.from_file(Rails.root.join('public/verification_skin.png'))
  end
end
```

Luckily, there's nothing fancy about getting a player's skin: they're just stored in
an S3 bucket with a filename matching their username. All I have to do is
download and load it into [ChunkyPNG][6] to compare it with the original. This
isn't incredibly fast, and I've considered other ways of doing it, namely, MD5
hexdigest comparison. However, that would have zero-tolerance of any
difference, and I wasn't sure if I could guarantee that the images would be
absolutely unchanged upon uploading them to Mojang. It's probably worth a try
though. The image diff just gives me the benefit of being able to set my own
tolerance.

So there you have it. With that little service class, I can verify that my users are who
they say they are and give them a little "verified" badge next to their name.

[1]: http://www.reddit.com/r/Civcraft/
[2]: https://github.com/Wolvereness/PhysicalShop
[3]: https://github.com/matthewbot/PrisonPearl
[4]: http://www.minecraftwiki.net/wiki/The_End
[5]: https://civtrade.herokuapp.com
[6]: https://github.com/wvanbergen/chunky_png
[7]: /images/prison-pearl.png "Trapped in The End"
