# SmsCountryApi

Ruby wrapper for the SMSCountry web service API.

## Building the gem

Before you can start building the gem, you need to have the latest bundler package
installed. This should be part of your normal Ruby tool chain.

The gem can be build with the standard sequence of commands run from the gem's
top-level source directory:

    bundle install

Thos command installs the necessary dependencies based on the Gemfile for
the gem. If dependencies are already installed (from a previous install), then the
sequence:

    bundle update

command will update the dependencies to the most recent versions that are compatible
with the conditions set in the Gemfile.

Note that `bundle update` won't install new/missing dependencies. `bundle install` does
that, and must be used if any new dependencies are added to the Gemfile. That's also why
it needs to be run initially before the gem can be built.

Once the dependencies are installed, the command:

    gem build SmsCountryApi.gemspec

will build and package the gem into `SmsCountryApi-X.Y.Z.gem` (where X.Y.Z is the
version number of the gem).

## Installation

Installation of the gem version X.Y.Z is done using the gemfile directly since this
gem isn't published to `rubygems.org`:

    gem install SmsCountryApi-X.Y.Z.gem

This gem depends on `rest-client` and `json`. When installing it you must
have those gems installed or be able to have those gems and their dependencies
installed automatically by the `gem` command.

For development you will need the development dependencies installed or able to
be installed automatically. The standard dependencies on `bundler`, `rake` and
`minitest` are needed as well as `yard` for documentation and `webmock` for testing.

Consult the gemspec for version requirements.

Documentation can be generated by issuing a `yard doc` command while in the top directory
of the gem. The documentation in HTML format will be placed in the `doc` directory, and
can be viewed in a browser by opening `doc/index.html`.

## SMSCountry API documentation

The documentation on the SMSCountry web service API is available at
[http://docs.smscountryapi.apiary.io/](http://docs.smscountryapi.apiary.io/).

## Usage

The canonical return from an API call method is an array. The first element will always
be a `StatusResponse` object containing the operation's success/failure flag, a message
describing the results of the operation and an API ID (a UUID). Only the success flag
is guaranteed to be populated, the message and API ID may be nil depending on the operation
and failure state. Some calls don't return more than that, and internal errors such as
illegal arguments produce a message but no API ID. Depending on the API call
there will be more elements containing results returned from the call. If the call failed,
those additional elements will be `nil`. The various methods all insure that their return
value contains the proper number of elements so you can use Ruby array unwrapping safely.
For methods that only return the status object, the Ruby idiom of appending a comma to a
single variable can be used. Thus:

    status, = client.call.terminate_call(call_uuid)

would get the status from a terminate-call operation which returns just the status object.

Client objects are created via two factory methods on the `SmsCountryApi::Client` class. The
`#create_client` method creates a client object based on the application's default configuration
for the SMSCountry service endpoint URL. The `#create_custom_client` method takes a protocol
string, hostname string and URL path prefix string as it's arguments and creates a client based
on the endpoint URL formed from those arguments. The application design will dictate which one will
be used. If desired the `SmsCountryApi::Client` class can be extended and the `#create_client`
method overridden to change how the configuration is accessed without requiring changes in the
application's main code.

#### Sending a single message and checking it's status

A basic program using the SMS interface to send a message and then retrieve it's details:

    require 'SmsCountryApi'

    # Create an API client object.
    client = SmsCountryApi::Client.create_client('authentication key', 'authentication token')

    # Send a single message to a fixed number.
    status, message_uuid = client.sms.send("91XXXXXXXXXX", "Example message.", sender_id: "SMSCountry",
                                           notify_url: "https://www.domainname.com/notifyurl")
    # If the operation failed, set no message UUID and log an error.
    unless status.success
        message_uuid = nil
        log.error(status.message.to_s)
    end

    # ...

    # If the message UUID is known, retrieve the details of the message and display them.
    unless message_uuid.nil?
        status, message_details = client.sms.get_details(message_uuid)
        if status.success
            puts message_details.status_time.to_s + ": " + message_details.status.to_s +
                 "  Cost: " + message_details.cost.to_s
        else
            log.error(status.message.to_s)
        end
    end

The `#to_s` method is used on the status and details accessors to insure there aren't any errors
if those fields are `nil`. While the methods do argument type checking on call arguments, results
are returned verbatim without any type checking and reflect exactly what was returned from the
web service. The caller is responsible for checking results for validity.

#### Sending a bulk message with a unique sender ID and retrieving details on those message_details

    require 'SmsCountryApi'

    # Create an API client object using the application default configuration.
    client = SmsCountryApi::Client.create_client('authentication key', 'authentication token')

    # Set up the list of numbers to send to, and send the message group.
    number_list = ["91XXXXXXAAAA", "91XXXXXXBBBB", "91XXXXXXCCCC"]
    status, batch_uuid, message_uuid_list = client.sms.bulk_send(number_list, "Example message.",
                                                                 sender_id: "Batch117",
                                                                 notify_url: "https://www.domainname.com/notifyurl")
    # If the operation succeeded, display the batch UUID. Otherwise, log an error.
    if status.success
        puts "Batch " + batch_uuid.to_s + " sent."
    else
        message_uuid_list = []
        log.error(status.message.to_s)
    end

    # ...

    # Get the collection of messages by sender ID and display their status.
    unless message_uuid_list.empty?
        puts "Batch " + batch_uuid.to_s + " status report"
        status, message_details = client.sms.get_collection(sender_id: "Batch117", limit: message_uuid_list.length)
        if status.success
            puts "Status not available for some messages." unless message_uuid_list.length == message_details.length
            # Go through the returned messages and display the status of each one.
            message_details.each do |detail|
                puts detail.status_time.to_s + ": " + detail.status.to_s + " Cost: " + detail.cost.to_s
            end
        else
            log.error(status.message.to_s)
        end
    end

When reporting the status details a message is displayed if the number of messages status was returned for
doesn't match the number of messages originally sent. When requesting status, the limit is set to the number
of messages sent so we should get all the status items. This works for short lists, larger collections or
collections where the number of returned items isn't known (eg. when requesting the status of all messages
sent in one day) the `limit` argument would be set to a reasonable value (eg. the number of records to be
displayed on one page) and the `offset` argument initially set to 0. The value of `offset` would then be
incremented by the value of `limit` and the `#get_collection` call repeated to get each subsequent set of
records until a call fails or returns 0 records. A successful call may return fewer than `limit` records,
for example the last successful call would be expected to return only the number of records remaining
regardless of how many were requested via `limit`, so code should be written based on the actual length of
the returned message details array and not on the number of records requested.
