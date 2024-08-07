ARG ELIXIR_VERSION
ARG ELIXIR_ERLANG_VERSION
ARG ELIXIR_ALPINE_VERSION
ARG _WORKDIR=/code

elixir.lint: elixir.dialyzer elixir.format elixir.credo

elixir.base:
    FROM docker.io/hexpm/elixir:${ELIXIR_VERSION}-erlang-${ELIXIR_ERLANG_VERSION}-alpine-${ELIXIR_ALPINE_VERSION}
    RUN apk add --no-cache git openssh-client
    ARG _WORKDIR
    WORKDIR ${_WORKDIR}

elixir.toolchain:
    @devshell
    FROM +elixir.base
    RUN apk add --no-cache git build-base
    RUN mix local.rebar --force && \
        mix local.hex --force

elixir.deps:
    FROM +elixir.toolchain
    COPY mix.exs mix.lock* ./
    RUN --mount=type=ssh mix deps.get --check-locked
    RUN mix deps.unlock --check-unused
    RUN MIX_ENV=dev mix deps.compile && \
        MIX_ENV=test mix deps.compile

elixir.compile:
    FROM +elixir.deps
    COPY config ./config
    COPY test ./test
    COPY lib ./lib
    COPY .*.exs ./
    RUN MIX_ENV=dev mix compile --warnings-as-errors && \
        MIX_ENV=test mix compile --warnings-as-errors

elixir.dialyzer-plt:
    FROM +elixir.deps
    # dialyzer plt under: _build/dev/dialyxir_*.plt*
    RUN mix dialyzer --plt

elixir.dialyzer:
    FROM +elixir.compile
    ARG _WORKDIR
    COPY --from=+elixir.dialyzer-plt ${_WORKDIR}/_build/dev/dialyxir_*.plt* ./_build/dev/
    RUN mix dialyzer --no-check

elixir.format:
    FROM +elixir.compile
    RUN mix format --check-formatted

elixir.credo:
    FROM +elixir.compile
    ARG ELIXIR_CREDO_OPTS="--strict --all"
    COPY mix.exs .credo.exs* ./
    RUN mix credo ${ELIXIR_CREDO_OPTS}

elixir.test:
    @output ${_WORKDIR}/cover
    FROM +elixir.compile
    ARG ELIXIR_TEST_CMD="coveralls.html"
    COPY coveralls.json* .
    RUN mix ${ELIXIR_TEST_CMD}

elixir.docs:
    @output ${_WORKDIR}/doc
    FROM +elixir.compile
    COPY README.md ./
    RUN mix docs --formatter=html

elixir.release:
    FROM +elixir.compile
    RUN mix release --path=_release

elixir.escript-build:
    FROM +elixir.compile
    RUN mix escript.build
    RUN ESCRIPT_NAME=$(mix run --no-start --no-compile --eval "IO.write(to_string(Mix.Project.config[:escript][:name] || Mix.Project.config[:app]))") && \
        cp "$ESCRIPT_NAME" escript

elixir.escript:
    FROM +elixir.base
    ARG _WORKDIR
    ARG ELIXIR_ESCRIPT_EXTRA_APK
    RUN apk add --no-cache ${ELIXIR_ESCRIPT_EXTRA_APK}
    COPY --from=+elixir.escript-build ${_WORKDIR}/escript /escript
    ENTRYPOINT [ "/escript" ]
