# asterisk-elk-grok
*Grok Filters For The Asterisk Debug Log*

This is a filter for logstash whenever asterisk logs are sent to a logstash instance for log visualtion mainly in the ELK stack.

This mainly encompases log lines that are in the following format:
```
[Apr 18 18:58:45] VERBOSE[24621][C-000001ed] pbx.c:     -- Executing [9999999999@public:3] Hangup("SIP/127.0.0.1-00000001", "") in new stack
[Apr 18 18:58:45] VERBOSE[24621][C-000001ed] pbx.c:   == Spawn extension (public, 9999999999, 3) exited non-zero on 'SIP/127.0.0.1-00000001'
```

It'll note the dialplan_extension, dialplan_context, dialplan_priority, and other information for each dialplan execution log message from the pbx.c module. Also it will note when mixmonitor recording begins and ends along with the mixmonitor channel name. Manager login and logoff messages are also filtered.

I've got this setup currently to include_lines on the prospector via filebeat that start with a '[' only.

## Asterisk Server Filebeat Example Configuration

```yaml
filebeat:
  prospectors:
    # Asterisk /var/log/asterisk/debug
    -
      paths:
        - /var/log/asterisk/debug
      input_type: log
      document_type: asterisk_debug
      include_lines: ["^[[]"]
```
## Logstash Server Grok Filter

```bash
filter {
    if [type] == "asterisk_debug" {

        if [message] =~ /^\[/ {
            # All asterisk logs will have the following fields
            # log_timestamp: timestamp from the debug log. In two possible formats in the date { match => line
            # log_level: DEBUG|NOTICE|wARNING|ERROR|VERBOSE|DTMF|FAX|SECURITY
            # thread_id: Channel thread ID within brackets after the log_level
            # call_thread_id: Call thread ID. Second set of brackets after log_level
            # module_name: name of the C module that outputs the log message
            # log_message: Everything after the module_name. Usually the full Asterisk CLI message
            # received_timestamp: ELK timestamp for when the log message was receieved
            # process_name: always "asterisk" for asterisk logs
            grok {
                match => { 
                    "message" => "\[%{SYSLOGTIMESTAMP:log_timestamp}\] +(?<log_level>(?i)(?:debug|notice|warning|error|verbose|dtmf|fax|security)(?-i))\[%{INT:thread_id}\](?:\[%{DATA:call_thread_id}\])? %{DATA:module_name}\:(?: +[=|-]{2})? %{GREEDYDATA:log_message}" 
                }
                add_field => [ "received_timestamp", "%{@timestamp}" ]
                add_field => [ "process_name", "asterisk" ]
            }
            date { match => [ "log_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ] }
    
    
            # For asterisk log messages that don't output anything after the module_name. Ex: app_dumpchan.c
            if ![log_message] {
                mutate {
                    add_field => {"log_message" => ""}
                }
            } # End default asterisk log fields
    
            # The pbx.c log lines include extra data about the dialplan execution and the originating channel.
            # This is mainly for dialplan lines. They start with "Executing" in the log_message.
            # dialplan_extension
            # dialplan_context
            # dialplan_priority
            # asterisk_app
            # asterisk_app_data
            # originating_channel
            if [log_message] =~ /^Executing/ and [module_name] == "pbx.c" {
                grok {
                    match => {
                        "log_message" => "Executing +\[%{DATA:dialplan_extension}@%{DATA:dialplan_context}:%{INT:dialplan_priority}\] +%{DATA:asterisk_app}\(\"%{DATA:originating_channel}\", \"%{GREEDYDATA:asterisk_app_data}\"\)"
                    }
                }
    
                # If the previous fields for the Executing dialplan lines were filled except for asterisk_app_data,
                # then add the field with a blank value so it's at least searchable
                # Examples include Hangup, Verbose, NoOp. Any asterisk app that could possible not have app_data
                if ![asterisk_app_data] {
                    mutate {
                        add_field => {"asterisk_app_data" => ""}
                    }
                }
    
            } # end asterisk default dialplan fields
    
            # For lines with '==' in them. They have a different format and usually indicate asterisk
            # low level core log messages
            if [message] =~ /\s+==\s+/ {
    
                # This is for the Spawn extension exited non zero lines so we can associated them with the dialplan
                # log messages in a different format
                if [module_name] and [module_name] =~ /^(pbx|app_stack)\.c/ and [log_message] =~ /^Spawn extension/ {
                    grok {
                        match => {
                            "log_message" => "^Spawn extension \(%{DATA:dialplan_context}, %{DATA:dialplan_extension}, %{DATA:dialplan_priority}\) exited (non-zero|\S+) on '${DATA:originating_channel}'"
                        }
                    }
                } # End spawn extension messages
    
                # Indicators for when mixmonitor starts and stops for a given channel
                if [module_name] and [module_name] == "app_mixmonitor.c" and [log_message] =~ /^(End|Begin) MixMonitor Recording/ {
                    grok {
                        match => {
                            "log_message" => "^(?<mixmonitor_action>(?:End|Begin)) MixMonitor Recording %{GREEDYDATA:mixmonitor_record_channel}"
                        }
                    }
                } # end mixmonitor start/stop data
    
                if [module_name] and [module_name] == "manager.c" {
                    if [log_message] =~ /^Manager '\S+' logged (on|off) from/ {
                        grok {
                            match => {
                                "log_message" => "Manager '%{DATA:manager_user}' (?<manager_user_status>logged (?:on|off)) from %{IP:manager_listen_ip}"
                            }
                        }
                    } # End manager logoff and logon action
                } # end manager.c actions (login/logoff/action execution)
    
                #if [module_name] and [module_name == "" {
                #}
            } # for messages that have '==' in them. They're different from the '--' messages
        } # end log lines that begin with '['


    } # end filter for type == asterisk_debug
} # end filter

```
