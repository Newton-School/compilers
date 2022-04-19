FROM judge0/compilers:1.4.0


# Check for latest version here: https://www.python.org/downloads
ENV PYTHON_VERSIONS \
      3.10.4

RUN set -xe && \
    for VERSION in $PYTHON_VERSIONS; do \
      curl -fSsL "https://www.python.org/ftp/python/$VERSION/Python-$VERSION.tar.xz" -o /tmp/python-$VERSION.tar.xz && \
      mkdir /tmp/python-$VERSION && \
      tar -xf /tmp/python-$VERSION.tar.xz -C /tmp/python-$VERSION --strip-components=1 && \
      rm /tmp/python-$VERSION.tar.xz && \
      cd /tmp/python-$VERSION && \
      ./configure \
        --prefix=/usr/local/python-$VERSION && \
      make -j$(nproc) && \
      make -j$(nproc) install && \
      rm -rf /tmp/*; \
    done

# Python for ML
RUN apt-get update && \
    /usr/local/python-3.10.4/bin/pip3 install \
        mlxtend \
        numpy \
        pandas \
        scikit-learn \
        pytest \
        scipy \
        matplotlib \
        gspread \
        pydrive \
        keras \
        scrapy \
        nltk \
        bs4 \
        seaborn \
        xgboost \
        opencv-python \
        Pillow \
        tensorflow && \
    rm -rf /var/lib/apt/lists/*
