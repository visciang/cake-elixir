ARG ELIXIR_VERSION
ARG ELIXIR_ERLANG_VERSION
ARG ELIXIR_ALPINE_VERSION
ARG _WORKDIR=/code

lint: dialyzer format credo

base:
    FROM docker.io/hexpm/elixir:${ELIXIR_VERSION}-erlang-${ELIXIR_ERLANG_VERSION}-alpine-${ELIXIR_ALPINE_VERSION}
    RUN apk add --no-cache git openssh-client
    ARG _WORKDIR
    WORKDIR ${_WORKDIR}

toolchain:
    @devshell
    FROM +base
    RUN apk add --no-cache git build-base
    RUN mix local.rebar --force && \
        mix local.hex --force

deps:
    FROM +toolchain
    COPY mix.exs mix.lock* ./
    RUN --mount=type=ssh mix deps.get --check-locked
    RUN mix deps.unlock --check-unused
    RUN MIX_ENV=dev mix deps.compile && \
        MIX_ENV=test mix deps.compile

compile:
    FROM +deps
    COPY config ./config
    COPY test ./test
    COPY lib ./lib
    COPY .*.exs ./
    RUN MIX_ENV=dev mix compile --warnings-as-errors && \
        MIX_ENV=test mix compile --warnings-as-errors

dialyzer_plt:
    FROM +deps
    # dialyzer plt under: _build/dev/dialyxir_*.plt*
    RUN mix dialyzer --plt

dialyzer:
    FROM +compile
    ARG _WORKDIR
    COPY --from=+dialyzer_plt ${_WORKDIR}/_build/dev/dialyxir_*.plt* ./_build/dev/
    RUN mix dialyzer --no-check

format:
    FROM +compile
    RUN mix format --check-formatted

credo:
    FROM +compile
    ARG ELIXIR_CREDO_OPTS="--strict --all"
    COPY mix.exs .credo.exs* ./
    RUN mix credo ${ELIXIR_CREDO_OPTS}

test:
    @output ${_WORKDIR}/cover
    FROM +compile
    ARG ELIXIR_TEST_CMD="coveralls.html"
    COPY coveralls.json* .
    RUN mix ${ELIXIR_TEST_CMD}

docs:
    @output ${_WORKDIR}/doc
    FROM +compile
    COPY README.md ./
    RUN mix docs --formatter=html

release:
    FROM +compile
    RUN mix release --path=_release

escript_build:
    FROM +compile
    RUN mix escript.build
    RUN ESCRIPT_NAME=$(mix run --no-start --no-compile --eval "IO.write(to_string(Mix.Project.config[:escript][:name] || Mix.Project.config[:app]))") && \
        cp "$ESCRIPT_NAME" escript

escript:
    FROM +base
    ARG _WORKDIR
    ARG ELIXIR_ESCRIPT_EXTRA_APK
    RUN apk add --no-cache ${ELIXIR_ESCRIPT_EXTRA_APK}
    COPY --from=+escript_build ${_WORKDIR}/escript /escript
    ENTRYPOINT [ "/escript" ]
