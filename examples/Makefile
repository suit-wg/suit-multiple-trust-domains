EXAMPLES = example1_process.suit example2_integrated.suit

.PHONY: all
all: $(EXAMPLES)

%.suit: %.diag
	diag2cbor.rb < $< > $@

.PHONY: clean
clean:
	$(RM) *.suit
