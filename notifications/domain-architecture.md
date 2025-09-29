# Notifications Domain Architecture

**Enterprise-grade multi-channel notification delivery system enabling rapid reunification through intelligent, timely notifications across Email, SMS, and WhatsApp.**

## Domain Purpose

Transform notification delivery from unreliable, single-channel systems to **intelligent, multi-channel orchestration** with enterprise resilience patterns, enabling critical Lost & Found communications to reach users through their preferred channels with guaranteed delivery and comprehensive retry mechanisms.

**Core Value**: Multi-channel delivery with intelligent routing ensures critical reunification notifications reach users reliably, while enterprise resilience patterns (circuit breakers, bulkheads) maintain system stability under high load.

## Domain Boundaries

**Bounded Context**: Multi-channel notification delivery orchestration
- Event-driven notification processing from Posts, Matcher, and User domains via Kafka
- Multi-channel delivery coordination (Email via Swoosh, SMS/WhatsApp via Twilio)
- User preference management with smart channel routing and quiet hours
- Enterprise resilience patterns with circuit breakers and bulkhead isolation
- Real-time operational dashboard with Phoenix LiveView
- Comprehensive retry logic with exponential backoff and dead letter handling

## Key Architecture Decisions

### 1. Elixir/OTP + Phoenix for Enterprise Resilience

**Decision**: Use Elixir with Phoenix framework and OTP supervision for the notifications service.

**Why**: Notifications are mission-critical requiring enterprise-grade fault tolerance:
- **Fault Tolerance**: OTP supervision trees ensure individual channel failures don't cascade
- **Concurrency**: Actor model enables thousands of concurrent notification deliveries
- **Real-time UI**: Phoenix LiveView provides real-time dashboard without JavaScript complexity
- **Hot Code Swapping**: Zero-downtime deployments for notification rule updates

**Technology Stack**:
```elixir
# Core Framework
phoenix           # Web framework with LiveView
broadway          # Event processing with backpressure
broadway_kafka    # Kafka consumer integration

# Delivery Channels
swoosh           # Email delivery abstraction
ex_twilio        # SMS and WhatsApp via Twilio API
tesla            # HTTP client for webhook integrations

# Enterprise Patterns
oban             # Background job processing with retry logic
cachex           # Distributed caching for user preferences
telemetry        # Observability and metrics collection

# Data Layer
ecto_sql         # Database abstraction with PostgreSQL
postgrex         # PostgreSQL driver
```

### 2. Event-Driven Architecture with Broadway

**Decision**: Use Broadway for Kafka event consumption with configurable concurrency and batching.

**Why**: Lost & Found notifications must be processed in real-time with backpressure handling:
- **Scalability**: Broadway stages enable independent scaling of event processing
- **Backpressure**: Automatic flow control prevents overwhelming downstream services
- **Reliability**: At-least-once delivery guarantees with explicit acknowledgment
- **Observability**: Built-in metrics and telemetry for monitoring event processing

**Broadway Configuration**:
```elixir
defmodule FnNotifications.Application.EventHandlers.PostsEventProcessor do
  use Broadway

  def start_link(_opts) do
    Broadway.start_link(__MODULE__,
      name: __MODULE__,
      producer: [
        module: {BroadwayKafka.Producer, kafka_config()},
        stages: 1
      ],
      processors: [
        default: [stages: 10, max_demand: 5]  # 10 concurrent processors
      ],
      batchers: [
        default: [batch_size: 20, batch_timeout: 1000]  # Batch for efficiency
      ]
    )
  end

  def handle_message(_processor, message, _context) do
    %{data: raw_data, headers: headers, key: key} = message.data

    # Decode external event
    external_event = Jason.decode!(raw_data)

    # Translate through anti-corruption layer
    case EventTranslator.translate(external_event) do
      {:ok, command} ->
        # Process notification command
        case NotificationService.send_notification(command) do
          {:ok, _notification} ->
            Logger.info("Processed notification", event_key: key)
            message
          {:error, reason} ->
            Logger.error("Failed to process notification",
              event_key: key, reason: reason)
            Broadway.Message.failed(message, reason)
        end
      {:error, translation_error} ->
        Logger.warn("Unknown event type",
          event_type: external_event["type"], error: translation_error)
        message  # Acknowledge unknown events to prevent reprocessing
    end
  end

  def handle_batch(_batcher, messages, _batch_info, _context) do
    # Batch processing for analytics or bulk operations
    Logger.info("Processed batch", count: length(messages))
    messages
  end

  defp kafka_config do
    [
      hosts: Application.get_env(:fn_notifications, :kafka_hosts),
      group_id: "fn-notifications-posts-consumer",
      topics: ["posts.events"],  # Handles post.created, post.updated, post.resolved, post.deleted
      offset_reset_policy: :earliest,
      max_bytes: 1_000_000,
      max_wait_time: 5_000
    ]
  end
end
```

### Matcher Event Processing

**Critical Addition**: The notification service also includes a dedicated `MatcherEventProcessor` for handling intelligent matching events from the fn-matcher service:

```elixir
defmodule FnNotifications.Application.EventHandlers.MatcherEventProcessor do
  use Broadway

  def start_link(_opts) do
    Broadway.start_link(__MODULE__,
      name: __MODULE__,
      producer: [
        module: {BroadwayKafka.Producer, matcher_kafka_config()},
        stages: 1
      ],
      processors: [
        default: [stages: 10, max_demand: 5]  # 10 concurrent processors
      ],
      batchers: [
        default: [batch_size: 20, batch_timeout: 1000]
      ]
    )
  end

  def handle_message(_processor, message, _context) do
    external_event = Jason.decode!(message.data.raw_data)

    # Translate matcher events through anti-corruption layer
    case EventTranslator.translate_matcher_event(external_event) do
      {:ok, notification_commands} ->
        # Process multiple notifications for match events (both parties)
        Enum.each(notification_commands, &NotificationService.send_notification/1)
        message
      {:error, reason} ->
        Logger.warning("Failed to translate matcher event", reason: reason)
        Broadway.Message.failed(message, reason)
    end
  end

  defp matcher_kafka_config do
    [
      hosts: Application.get_env(:fn_notifications, :kafka_hosts),
      group_id: "fn-notifications-matcher-consumer",
      topics: ["posts.matching"],  # Events from fn-matcher service
      offset_reset_policy: :earliest,
      max_bytes: 1_000_000,
      max_wait_time: 5_000
    ]
  end
end
```

**Matcher Events Handled**:
- **`post.matched`**: Notifies both reporter and matcher via email about potential match
- **`post.claimed`**: Sends urgent SMS to reporter, confirmation email to claimer
- **`match.expired`**: Notifies both parties via email when match expires without action

**Business Impact**: These events enable the core Lost & Found reunification workflow - without them, users would never be notified of critical matching activities.

### 3. Multi-Channel Delivery with Adapter Pattern

**Decision**: Implement channel-specific adapters with unified interface for Email, SMS, and WhatsApp.

**Why**: Different channels have unique APIs and failure modes requiring specialized handling:
- **Consistency**: Unified interface enables consistent notification processing
- **Flexibility**: Easy addition of new channels (Push, Slack, etc.)
- **Resilience**: Channel-specific circuit breakers prevent cascade failures
- **Testing**: Isolated adapters enable thorough testing of delivery logic

**Adapter Architecture**:
```elixir
# Unified delivery behavior
defmodule FnNotifications.Domain.Repositories.DeliveryAdapterBehavior do
  @callback deliver(notification :: Notification.t(), preferences :: UserPreferences.t()) ::
    {:ok, :delivered} | {:error, term()}
  @callback supports_channel?(channel :: atom()) :: boolean()
  @callback get_delivery_cost(notification :: Notification.t()) :: non_neg_integer()
end

# Email adapter with Swoosh
defmodule FnNotifications.Infrastructure.Adapters.EmailAdapter do
  @behaviour FnNotifications.Domain.Repositories.DeliveryAdapterBehavior

  def deliver(notification, user_preferences) do
    if user_preferences.email do
      email = Swoosh.Email.new()
      |> to({user_preferences.name || "User", user_preferences.email})
      |> from({"Findly Now", Application.get_env(:fn_notifications, :from_email)})
      |> subject(notification_subject(notification))
      |> text_body(notification.body)
      |> html_body(render_html_template(notification, user_preferences))

      case FnNotifications.Mailer.deliver(email) do
        {:ok, _} -> {:ok, :delivered}
        {:error, reason} -> {:error, {:email_delivery_failed, reason}}
      end
    else
      {:error, :no_email_address}
    end
  end

  def supports_channel?(:email), do: true
  def supports_channel?(_), do: false

  def get_delivery_cost(_notification), do: 1  # Cost units for rate limiting

  defp notification_subject(%{notification_type: :post_confirmation}),
    do: "Your item has been reported - Findly Now"
  defp notification_subject(%{notification_type: :urgent_claim}),
    do: "URGENT: Someone wants to claim your item!"
  defp notification_subject(%{notification_type: :match_detected}),
    do: "Possible match found for your item"
  defp notification_subject(_),
    do: "Notification from Findly Now"

  defp render_html_template(notification, user_preferences) do
    Phoenix.View.render_to_string(
      FnNotificationsWeb.EmailView,
      "#{notification.notification_type}.html",
      notification: notification,
      user: user_preferences,
      base_url: Application.get_env(:fn_notifications, :web_base_url)
    )
  end
end

# SMS adapter with Twilio
defmodule FnNotifications.Infrastructure.Adapters.SmsAdapter do
  @behaviour FnNotifications.Domain.Repositories.DeliveryAdapterBehavior

  def deliver(notification, user_preferences) do
    if user_preferences.phone do
      case ExTwilio.Message.create(
        from: Application.get_env(:ex_twilio, :phone_number),
        to: user_preferences.phone,
        body: format_sms_message(notification)
      ) do
        {:ok, %{status: status}} when status in ["sent", "queued"] ->
          {:ok, :delivered}
        {:ok, %{status: failed_status}} ->
          {:error, {:sms_failed, failed_status}}
        {:error, reason} ->
          {:error, {:twilio_error, reason}}
      end
    else
      {:error, :no_phone_number}
    end
  end

  def supports_channel?(:sms), do: true
  def supports_channel?(_), do: false

  def get_delivery_cost(_notification), do: 5  # Higher cost for SMS

  defp format_sms_message(notification) do
    # SMS has character limits, so we format appropriately
    case notification.notification_type do
      :urgent_claim ->
        "URGENT: Someone wants to claim your lost item! Check Findly Now app for details."
      :match_detected ->
        "Possible match found for your #{extract_item_type(notification)}! View: #{short_url(notification)}"
      :post_confirmation ->
        "Your #{extract_item_type(notification)} has been reported on Findly Now. We'll notify you of any matches."
      _ ->
        String.slice(notification.body, 0, 140) <> "..."
    end
  end
end

# WhatsApp adapter (extends SMS adapter)
defmodule FnNotifications.Infrastructure.Adapters.WhatsAppAdapter do
  @behaviour FnNotifications.Domain.Repositories.DeliveryAdapterBehavior

  def deliver(notification, user_preferences) do
    if user_preferences.phone && whatsapp_enabled?(user_preferences) do
      case ExTwilio.Message.create(
        from: "whatsapp:" <> Application.get_env(:ex_twilio, :whatsapp_number),
        to: "whatsapp:" <> user_preferences.phone,
        body: format_whatsapp_message(notification),
        media_url: get_media_urls(notification)
      ) do
        {:ok, %{status: status}} when status in ["sent", "queued"] ->
          {:ok, :delivered}
        {:error, reason} ->
          {:error, {:whatsapp_error, reason}}
      end
    else
      {:error, :whatsapp_not_available}
    end
  end

  def supports_channel?(:whatsapp), do: true
  def supports_channel?(_), do: false

  def get_delivery_cost(_notification), do: 3  # Medium cost for WhatsApp

  defp whatsapp_enabled?(user_preferences) do
    get_in(user_preferences.channel_preferences, ["whatsapp", "enabled"]) == true
  end
end
```

### 4. Enterprise Resilience with Circuit Breakers and Bulkheads

**Decision**: Implement circuit breaker and bulkhead patterns to prevent cascade failures across delivery channels.

**Why**: External service failures must not impact overall notification system:
- **Isolation**: Circuit breakers prevent failures from cascading between channels
- **Resource Protection**: Bulkheads limit resource consumption per channel
- **Graceful Degradation**: Failed channels don't prevent delivery via working channels
- **Recovery**: Automatic recovery when external services return to health

**Circuit Breaker Implementation**:
```elixir
defmodule FnNotifications.Domain.Services.CircuitBreakerService do
  use GenServer

  @failure_threshold 5
  @timeout_ms 60_000
  @reset_timeout_ms 300_000

  defstruct [
    :service_name,
    :state,           # :closed | :open | :half_open
    :failure_count,
    :success_count,
    :last_failure_time,
    :last_success_time
  ]

  def start_link(service_name) do
    GenServer.start_link(__MODULE__, service_name, name: via_tuple(service_name))
  end

  def call(service_name, function, timeout \\ 5000) do
    case get_state(service_name) do
      :closed ->
        execute_and_monitor(service_name, function, timeout)
      :open ->
        if should_attempt_reset?(service_name) do
          transition_to_half_open(service_name)
          execute_and_monitor(service_name, function, timeout)
        else
          {:error, :circuit_open}
        end
      :half_open ->
        execute_with_test(service_name, function, timeout)
    end
  end

  def get_state(service_name) do
    GenServer.call(via_tuple(service_name), :get_state)
  end

  # GenServer callbacks
  def init(service_name) do
    state = %__MODULE__{
      service_name: service_name,
      state: :closed,
      failure_count: 0,
      success_count: 0,
      last_failure_time: nil,
      last_success_time: nil
    }
    {:ok, state}
  end

  def handle_call(:get_state, _from, state) do
    {:reply, state.state, state}
  end

  def handle_call({:record_success}, _from, state) do
    new_state = %{state |
      state: :closed,
      failure_count: 0,
      success_count: state.success_count + 1,
      last_success_time: :os.system_time(:millisecond)
    }
    {:reply, :ok, new_state}
  end

  def handle_call({:record_failure}, _from, state) do
    failure_count = state.failure_count + 1
    now = :os.system_time(:millisecond)

    new_state = if failure_count >= @failure_threshold do
      %{state |
        state: :open,
        failure_count: failure_count,
        last_failure_time: now
      }
    else
      %{state |
        failure_count: failure_count,
        last_failure_time: now
      }
    end

    {:reply, :ok, new_state}
  end

  defp execute_and_monitor(service_name, function, timeout) do
    task = Task.async(fn -> function.() end)

    case Task.yield(task, timeout) || Task.shutdown(task) do
      {:ok, {:ok, result}} ->
        record_success(service_name)
        {:ok, result}
      {:ok, {:error, reason}} ->
        record_failure(service_name)
        {:error, reason}
      nil ->
        record_failure(service_name)
        {:error, :timeout}
    end
  end

  defp record_success(service_name) do
    GenServer.call(via_tuple(service_name), {:record_success})
  end

  defp record_failure(service_name) do
    GenServer.call(via_tuple(service_name), {:record_failure})
  end

  defp via_tuple(service_name) do
    {:via, Registry, {FnNotifications.CircuitBreakerRegistry, service_name}}
  end
end

# Bulkhead resource isolation
defmodule FnNotifications.Domain.Services.BulkheadService do
  use GenServer

  @pools %{
    email: %{max_concurrency: 10, timeout: 30_000},
    sms: %{max_concurrency: 5, timeout: 15_000},
    whatsapp: %{max_concurrency: 5, timeout: 15_000}
  }

  def start_link(_opts) do
    GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  end

  def execute(channel, function) do
    pool_config = Map.get(@pools, channel)

    if pool_config do
      case acquire_slot(channel, pool_config) do
        {:ok, slot_id} ->
          try do
            Task.async(fn ->
              result = function.()
              release_slot(channel, slot_id)
              result
            end)
            |> Task.await(pool_config.timeout)
          catch
            :exit, {:timeout, _} ->
              release_slot(channel, slot_id)
              {:error, :timeout}
          rescue
            error ->
              release_slot(channel, slot_id)
              {:error, error}
          end
        {:error, :pool_exhausted} ->
          {:error, :too_many_requests}
      end
    else
      {:error, :unknown_channel}
    end
  end

  defp acquire_slot(channel, _config) do
    GenServer.call(__MODULE__, {:acquire_slot, channel})
  end

  defp release_slot(channel, slot_id) do
    GenServer.cast(__MODULE__, {:release_slot, channel, slot_id})
  end
end
```

### 5. User Preferences with Smart Channel Routing

**Decision**: Store user preferences separately from notifications with intelligent channel selection.

**Why**: Contact information and preferences are user domain concerns, not notification concerns:
- **Domain Separation**: Contact info belongs to user preferences, not notification metadata
- **Caching Strategy**: User preferences cached for performance with TTL invalidation
- **Smart Routing**: Context-aware channel selection based on notification urgency and user preferences
- **Privacy**: Contact information centralized with proper access controls

**User Preferences Architecture**:
```elixir
defmodule FnNotifications.Domain.Entities.UserPreferences do
  @type t :: %__MODULE__{
    user_id: String.t(),
    global_enabled: boolean(),
    email: String.t() | nil,
    phone: String.t() | nil,
    timezone: String.t(),
    language: String.t(),
    channel_preferences: map(),
    quiet_hours: map(),
    inserted_at: DateTime.t(),
    updated_at: DateTime.t()
  }

  defstruct [
    :user_id, :global_enabled, :email, :phone, :timezone, :language,
    :channel_preferences, :quiet_hours, :inserted_at, :updated_at
  ]

  def create(user_id, email \\ nil, phone \\ nil) do
    %__MODULE__{
      user_id: user_id,
      global_enabled: true,
      email: email,
      phone: phone,
      timezone: "UTC",
      language: "en",
      channel_preferences: default_channel_preferences(),
      quiet_hours: %{},
      inserted_at: DateTime.utc_now(),
      updated_at: DateTime.utc_now()
    }
  end

  def update_contact_info(preferences, email, phone) do
    %{preferences |
      email: email,
      phone: phone,
      updated_at: DateTime.utc_now()
    }
    |> validate_contact_channels()
  end

  def enable_channel(preferences, channel) do
    new_channel_prefs = put_in(
      preferences.channel_preferences,
      [Atom.to_string(channel), "enabled"],
      true
    )
    %{preferences |
      channel_preferences: new_channel_prefs,
      updated_at: DateTime.utc_now()
    }
  end

  def disable_channel(preferences, channel) do
    new_channel_prefs = put_in(
      preferences.channel_preferences,
      [Atom.to_string(channel), "enabled"],
      false
    )
    %{preferences |
      channel_preferences: new_channel_prefs,
      updated_at: DateTime.utc_now()
    }
  end

  def set_quiet_hours(preferences, start_hour, end_hour, timezone \\ nil) do
    tz = timezone || preferences.timezone
    quiet_hours = %{
      "enabled" => true,
      "start_hour" => start_hour,
      "end_hour" => end_hour,
      "timezone" => tz
    }
    %{preferences |
      quiet_hours: quiet_hours,
      updated_at: DateTime.utc_now()
    }
  end

  def in_quiet_hours?(preferences) do
    if preferences.quiet_hours["enabled"] do
      now = DateTime.now!(preferences.quiet_hours["timezone"] || "UTC")
      current_hour = now.hour
      start_hour = preferences.quiet_hours["start_hour"]
      end_hour = preferences.quiet_hours["end_hour"]

      if start_hour <= end_hour do
        current_hour >= start_hour && current_hour < end_hour
      else
        # Quiet hours span midnight
        current_hour >= start_hour || current_hour < end_hour
      end
    else
      false
    end
  end

  defp default_channel_preferences do
    %{
      "email" => %{"enabled" => true},
      "sms" => %{"enabled" => false},
      "whatsapp" => %{"enabled" => false}
    }
  end

  defp validate_contact_channels(preferences) do
    # Disable channels if no contact info available
    new_prefs = preferences.channel_preferences
    |> maybe_disable_channel("email", is_nil(preferences.email))
    |> maybe_disable_channel("sms", is_nil(preferences.phone))
    |> maybe_disable_channel("whatsapp", is_nil(preferences.phone))

    %{preferences | channel_preferences: new_prefs}
  end

  defp maybe_disable_channel(channel_prefs, channel, should_disable) do
    if should_disable do
      put_in(channel_prefs, [channel, "enabled"], false)
    else
      channel_prefs
    end
  end
end

# Smart channel routing service
defmodule FnNotifications.Application.Services.ChannelRoutingService do
  def select_channels(notification_type, user_preferences) do
    base_channels = case notification_type do
      :urgent_claim ->
        # Always use multiple channels for urgent notifications
        [:sms, :email]
      :match_detected ->
        # SMS if enabled, otherwise email
        if channel_enabled?(user_preferences, :sms) do
          [:sms, :email]
        else
          [:email]
        end
      :post_confirmation ->
        # User's primary channel only
        [get_primary_channel(user_preferences)]
      :post_resolved ->
        # Email for success stories (not urgent)
        [:email]
      _ ->
        [:email]  # Default fallback
    end

    # Filter by user preferences and availability
    base_channels
    |> Enum.filter(&channel_enabled?(user_preferences, &1))
    |> Enum.filter(&contact_info_available?(user_preferences, &1))
    |> respect_quiet_hours(user_preferences, notification_type)
  end

  defp channel_enabled?(preferences, channel) do
    get_in(preferences.channel_preferences, [Atom.to_string(channel), "enabled"]) == true
  end

  defp contact_info_available?(preferences, :email), do: !is_nil(preferences.email)
  defp contact_info_available?(preferences, :sms), do: !is_nil(preferences.phone)
  defp contact_info_available?(preferences, :whatsapp), do: !is_nil(preferences.phone)

  defp get_primary_channel(preferences) do
    cond do
      channel_enabled?(preferences, :email) -> :email
      channel_enabled?(preferences, :sms) -> :sms
      channel_enabled?(preferences, :whatsapp) -> :whatsapp
      true -> :email  # Fallback
    end
  end

  defp respect_quiet_hours(channels, preferences, notification_type) do
    # Allow urgent notifications during quiet hours
    if notification_type == :urgent_claim or not UserPreferences.in_quiet_hours?(preferences) do
      channels
    else
      # During quiet hours, only use email
      Enum.filter(channels, &(&1 == :email))
    end
  end
end
```

### 6. Anti-Corruption Layer for External Events

**Decision**: Implement event translation layer to protect domain from external schema changes.

**Why**: External domains evolve independently requiring protection from breaking changes:
- **Domain Protection**: Internal domain models remain stable despite external changes
- **Schema Evolution**: External event format changes don't break notification processing
- **Business Logic Isolation**: Translation layer contains external complexity
- **Testing**: Domain logic can be tested independently of external event formats

**Event Translation Architecture**:
```elixir
defmodule FnNotifications.Application.AntiCorruption.EventTranslator do
  @moduledoc """
  Translates external domain events into internal notification commands.
  Protects the notifications domain from external schema changes.
  """

  def translate(%{"type" => "post.created", "data" => data} = event) do
    {:ok, %SendNotificationCommand{
      user_id: data["user_id"],
      notification_type: :post_confirmation,
      title: "Your item has been reported",
      body: build_post_confirmation_message(data),
      channels: [:email],  # Confirmations via email only
      metadata: %{
        post_id: data["post_id"],
        post_type: data["post_type"],
        location: data["location"],
        deduplication_key: "post_confirmation_#{data["post_id"]}"
      },
      scheduled_at: nil  # Send immediately
    }}
  end

  def translate(%{"type" => "post.matched", "data" => data} = event) do
    {:ok, %SendNotificationCommand{
      user_id: data["user_id"],
      notification_type: :match_detected,
      title: "Possible match found!",
      body: build_match_detected_message(data),
      channels: [:email, :sms],  # Multi-channel for matches
      metadata: %{
        post_id: data["post_id"],
        match_id: data["match_id"],
        confidence_score: data["confidence_score"],
        deduplication_key: "match_detected_#{data["match_id"]}"
      },
      scheduled_at: nil
    }}
  end

  def translate(%{"type" => "post.claimed", "data" => data} = event) do
    {:ok, %SendNotificationCommand{
      user_id: data["user_id"],
      notification_type: :urgent_claim,
      title: "URGENT: Someone wants to claim your item!",
      body: build_urgent_claim_message(data),
      channels: [:sms, :email, :whatsapp],  # All channels for urgent
      metadata: %{
        post_id: data["post_id"],
        claim_id: data["claim_id"],
        claimer_contact: data["claimer_contact"],
        deduplication_key: "urgent_claim_#{data["claim_id"]}"
      },
      scheduled_at: nil  # Immediate delivery for urgent notifications
    }}
  end

  def translate(%{"type" => "post.resolved", "data" => data} = event) do
    {:ok, %SendNotificationCommand{
      user_id: data["user_id"],
      notification_type: :post_resolved,
      title: "Great news! Your item was resolved",
      body: build_resolution_message(data),
      channels: [:email],  # Success stories via email
      metadata: %{
        post_id: data["post_id"],
        resolution_type: data["resolution_type"],
        deduplication_key: "post_resolved_#{data["post_id"]}"
      },
      scheduled_at: nil
    }}
  end

  def translate(%{"type" => "user.registered", "data" => data} = event) do
    {:ok, %SendNotificationCommand{
      user_id: data["user_id"],
      notification_type: :welcome,
      title: "Welcome to Findly Now!",
      body: build_welcome_message(data),
      channels: [:email],
      metadata: %{
        organization_id: data["organization_id"],
        deduplication_key: "welcome_#{data["user_id"]}"
      },
      scheduled_at: nil
    }}
  end

  def translate(%{"type" => unknown_type} = event) do
    Logger.warn("Unknown event type received", event_type: unknown_type, event: event)
    {:error, :unknown_event_type}
  end

  # Message builders with business context
  defp build_post_confirmation_message(data) do
    item_type = if data["post_type"] == "lost", do: "lost item", else: "found item"
    location_name = extract_location_name(data["location"])

    """
    Your #{item_type} "#{data["title"]}" has been successfully reported on Findly Now.

    📍 Location: #{location_name}
    🔍 We're actively searching for matches in the area

    We'll notify you immediately if we find any potential matches.
    You can view and manage your post at: https://app.findlynow.com/posts/#{data["post_id"]}

    The faster we act, the better the chance of recovery!

    - The Findly Now Team
    """
  end

  defp build_match_detected_message(data) do
    confidence_percent = round((data["confidence_score"] || 0.0) * 100)

    """
    Great news! We found a potential match for your item with #{confidence_percent}% confidence.

    🎯 Match Details:
    • Confidence Score: #{confidence_percent}%
    • Location: Similar area to your report
    • Time: Recent posting

    👀 Review Match: https://app.findlynow.com/matches/#{data["match_id"]}

    Please review the match and confirm if this looks like your item.
    Quick action increases the chance of successful recovery!

    - The Findly Now Team
    """
  end

  defp build_urgent_claim_message(data) do
    """
    🚨 URGENT: Someone wants to claim your lost item!

    A person has initiated the claiming process for "#{data["title"]}".
    This could be the person who found your item!

    ⚡ Immediate Actions Required:
    1. Review the claim details
    2. Verify the claimer's information
    3. Coordinate a safe pickup location

    🔗 Respond Now: https://app.findlynow.com/claims/#{data["claim_id"]}

    Time is critical for successful recovery. Please respond as soon as possible.

    - The Findly Now Team
    """
  end

  defp build_resolution_message(data) do
    resolution_text = case data["resolution_type"] do
      "claimed" -> "successfully claimed and returned to you"
      "found" -> "marked as found"
      "expired" -> "automatically expired"
      _ -> "resolved"
    end

    """
    🎉 Excellent news! Your item "#{data["title"]}" has been #{resolution_text}.

    Thank you for using Findly Now to help reunite you with your belongings.

    💝 Help Others: Consider sharing your success story to encourage others to use our platform.

    📝 Feedback: How was your experience? We'd love to hear from you:
    https://app.findlynow.com/feedback

    Together, we're building a world where nothing stays lost forever.

    - The Findly Now Team
    """
  end

  defp build_welcome_message(data) do
    org_name = data["organization_name"] || "your organization"

    """
    Welcome to Findly Now! 🎉

    You're now part of #{org_name}'s lost & found community. Here's how to get started:

    📱 Report Items: Use our mobile app for quick, photo-first reporting
    🔍 Smart Search: Our AI-powered matching connects lost items with their owners
    📧 Stay Informed: We'll notify you of matches and updates automatically

    🚀 Get Started: https://app.findlynow.com/onboarding

    Questions? Our team is here to help: support@findlynow.com

    Together, let's reunite people with their belongings!

    - The Findly Now Team
    """
  end

  defp extract_location_name(%{"address" => address}) when is_binary(address), do: address
  defp extract_location_name(%{"lat" => lat, "lng" => lng}), do: "#{lat}, #{lng}"
  defp extract_location_name(_), do: "Unknown location"
end
```

## Domain Model

**Aggregate Root**: `Notification`
- Manages delivery lifecycle with status transitions (pending → sent → delivered/failed)
- Enforces retry limits and exponential backoff timing
- Validates channel compatibility with user preferences
- Publishes domain events for delivery tracking and analytics

**Entities**:
- `UserPreferences` - Contact information and notification settings per user
- `DeliveryAttempt` - Individual delivery attempts with timing and error tracking

**Value Objects**:
- `NotificationChannel` - Email, SMS, WhatsApp enumeration with validation
- `NotificationStatus` - Pending, Sent, Delivered, Failed status transitions
- `DeliveryMetadata` - Template variables, tracking IDs, and external references

**Domain Services**:
- `CircuitBreakerService` - Prevents cascade failures across delivery channels
- `BulkheadService` - Resource isolation and concurrency limiting
- `ChannelRoutingService` - Smart channel selection based on notification type and user preferences
- `NotificationDeduplicationService` - Prevents duplicate notifications using deduplication keys

## Performance Targets

- **Event Processing**: <100ms from Kafka consumption to notification creation
- **Multi-Channel Delivery**: <5 seconds for all enabled channels
- **User Preference Loading**: <50ms with caching, <200ms without cache
- **Circuit Breaker Response**: <10ms for open circuit, <5s for half-open test

## Integration Points

**Consumes Events**:
- `post.created` → Posts domain (confirmation notifications)
- `post.matched` → Matcher domain (match alert notifications)
- `post.claimed` → Matcher domain (urgent claim notifications)
- `post.resolved` → Posts domain (success story notifications)
- `user.registered` → User domain (welcome notifications)

**External Services**:
- **Confluent Cloud Kafka**: Event consumption with Broadway consumer groups
- **Swoosh + Cloud Email**: Email delivery via SendGrid/Mailgun/SES
- **Twilio**: SMS and WhatsApp Business API for messaging
- **PostgreSQL**: Notification storage with JSONB metadata
- **Cachex**: User preferences caching with TTL expiration

**Real-time UI**:
- **Phoenix LiveView Dashboard**: Real-time notification monitoring and analytics
- **Phoenix PubSub**: Internal event broadcasting for live updates
- **Telemetry**: Metrics collection for Prometheus/Grafana monitoring

## Resilience Patterns Implementation

**Circuit Breaker Configuration**:
- **Email Circuit**: 5 failures trigger open state, 5-minute reset timeout
- **SMS Circuit**: 3 failures trigger open state, 10-minute reset timeout
- **WhatsApp Circuit**: 3 failures trigger open state, 10-minute reset timeout

**Bulkhead Resource Limits**:
- **Email Pool**: 10 concurrent deliveries, 30-second timeout
- **SMS Pool**: 5 concurrent deliveries, 15-second timeout
- **WhatsApp Pool**: 5 concurrent deliveries, 15-second timeout

**Retry Logic**:
- **Exponential Backoff**: 1s, 2s, 4s, 8s, 16s, 32s intervals
- **Maximum Retries**: 3 attempts for transient failures
- **Dead Letter Queue**: Failed notifications stored for manual review
- **Jitter**: Random delay (±20%) prevents thundering herd

## Business Rules Enforced

- **Contact Information Isolation**: Email/phone stored only in user_preferences table
- **Channel Availability**: Notifications only sent to enabled channels with valid contact info
- **Quiet Hours Respect**: Non-urgent notifications delayed during user's quiet hours
- **Deduplication**: Identical notifications within 1 hour automatically deduplicated
- **User Consent**: Global disable switch immediately stops all notifications
- **Retry Limits**: Maximum 3 retry attempts to prevent infinite loops
- **Urgent Override**: Urgent notifications (claims) bypass quiet hours and use all channels

---

*This domain focuses purely on reliable, multi-channel notification delivery with enterprise resilience patterns. For system-wide architecture decisions, see [../ARCHITECTURE.md](../ARCHITECTURE.md). For detailed API specifications, see [api-documentation.md](api-documentation.md).*