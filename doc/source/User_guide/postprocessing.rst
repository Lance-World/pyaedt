Postprocessing
==============

Postprocessing is essential in simulation. PyAEDT offers the capability to read solutions and visualize results
both within AEDT and externally using the `pyvista <https://www.pyvista.org/>`_
and `matplotlib <https://matplotlib.org/>`_ packages.

To use PyAEDT to create a report in AEDT, you can follow this general structure:

.. code:: python

    from ansys.aedt.core import Hfss
    hfss = Hfss()
    hfss.analyze()
    hfss.post.create_report(["db(S11)", "db(S12)"])


.. image:: ../Resources/sparams.jpg
  :width: 800
  :alt: AEDT report


You can also generate reports in Matplotlib:

.. code:: python

    from ansys.aedt.core import Hfss
    hfss = Hfss()
    hfss.analyze()

    traces_to_plot = hfss.get_traces_for_plot(second_element_filter="P1*")
    report = hfss.post.create_report(traces_to_plot)  # Creates a report in HFSS
    solution = report.get_solution_data()
    plt = solution.plot(solution.expressions)  # Matplotlib axes object.


.. image:: ../Resources/sparams_w_matplotlib.jpg
  :width: 800
  :alt: S-Parameters report created with Matplotlib


You can use PyAEDT to plot any kind of report available in the Electronics Desktop interface.
To access all available category, use the ``reports_by_category`` class.

.. code:: python

    from ansys.aedt.core import Hfss
    hfss = Hfss()
    hfss.analyze()
    # Create a 3D far field
    new_report = hfss.post.reports_by_category.far_field(expressions="db(RealizedGainTotal)",
                                                         setup=hfss.nominal_adaptive)


You can plot the field plot directly in HFSS and export it to image files.

.. code:: python

    from ansys.aedt.core import Hfss
    hfss = Hfss()
    hfss.analyze()

    cutlist = ["Global:XY"]
    setup_name = hfss.existing_analysis_sweeps[0]
    quantity_name = "ComplexMag_E"
    intrinsic = {"Freq": "5GHz", "Phase": "180deg"}
    # Create a field plot
    plot1 = hfss.post.create_fieldplot_cutplane(objlist=cutlist,
                                                quantityName=quantity_name,
                                                setup=setup_name,
                                                intrinsics=intrinsic)


.. image:: ../Resources/field_plot.png
  :width: 800
  :alt: Postprocessing features


PyAEDT leverages PyVista to export and plot fields outside AEDT, generating images and animations:

.. code:: python

    from ansys.aedt.core import Hfss
    hfss = Hfss()
    hfss.analyze()
    cutlist = ["Global:XY"]
    setup_name = hfss.existing_analysis_sweeps[0]
    quantity_name = "ComplexMag_E"
    intrinsic = {"Freq": "5GHz", "Phase": "180deg"}
    hfss.logger.info("Generating the image")
    plot_obj = hfss.post.plot_field(
            quantity="Mag_E",
            objects_list=cutlist,
            plot_type="CutPlane",
            setup=setup_name,
            intrinsics=intrinsic
        )


.. image:: ../Resources/pyvista_plot.jpg
  :width: 800
  :alt: Postprocessing features


PyAEDT includes a powerful class to generate a PDF report that is based on the Python ``fpdf2`` package.

To see the capabilities offered by PyAEDT, download a sample PDF report that uses this class:

:download:`PDF report example <../Resources/report_example.pdf>`

This code creates the previous PDF report:

.. code:: python

    from ansys.aedt.core.visualization.plot.pdf import AnsysReport
    import os
    report = AnsysReport()
    report.aedt_version = "2025R1"
    report.template_name = "AnsysTemplate"
    report.project_name = "Coaxial1"
    report.design_name = "Design2"
    report.template_data.font = "times"
    report.create()
    report.add_chapter("Chapter 1")
    report.add_sub_chapter("C1")
    report.add_text("Hello World.\nlorem ipsum....")
    report.add_text("ciao2", True, True)
    report.add_empty_line(2)
    report.add_page_break()
    report.add_chapter("Chapter 2")
    report.add_sub_chapter("Charts")
    local_path = r'C:\result'
    report.add_section(portrait=False, page_format="a3")
    report.add_image(os.path.join(local_path, "return_loss.jpg"), width=400, caption="S-Parameters")
    report.add_section(portrait=False, page_format="a5")
    report.add_table("MyTable", [["x", "y"], ["0", "1"], ["2", "3"], ["10", "20"]])
    report.add_section()
    report.add_chart([0, 1, 2, 3, 4, 5], [10, 20, 4, 30, 40, 12], "Freq", "Val", "MyTable")
    report.add_toc()
    report.save_pdf(r'c:\temp', "report_example.pdf")


.. image:: ../Resources/pdf_report.jpg
  :width: 800
  :alt: PDF report output example


Apply templates and annotations to existing Circuit reports
-----------------------------------------------------------

You can reuse report templates, add markers, and insert trace characteristics on
every existing Circuit report by leveraging the high-level report objects
available in :mod:`ansys.aedt.core`. The following helper shows a typical
workflow:

.. code:: python

    from pathlib import Path

    from ansys.aedt.core import Circuit, Desktop


    def export_circuit_reports(
        project_path,
        design_name,
        template_file=None,
        output_directory=None,
        marker_x="16GHz",
        marker_name="MX1",
        trace_characteristic="YAtXVal",
        trace_arguments=None,
        trace_solution_range=None,
        image_size=(1920, 1080),
    ):
        """Apply templates and export all Circuit reports as images."""

        trace_arguments = trace_arguments or []
        trace_solution_range = trace_solution_range or ["Full"]

        with Desktop(new_desktop_session=False, specified_version="2025.2", close_on_exit=False):
            with Circuit(project=project_path, design=design_name, specified_version="2025.2") as circuit:
                post = circuit.post

                export_dir = Path(output_directory or Path(project_path).parent / "CircuitReportImages")
                export_dir.mkdir(parents=True, exist_ok=True)

                for plot in post.plots:
                    if template_file:
                        plot.apply_report_template(template_file, property_type="All")

                    if marker_x is not None:
                        plot.add_cartesian_x_marker(marker_x, name=marker_name)

                    if trace_characteristic:
                        plot.add_trace_characteristics(
                            trace_characteristic,
                            arguments=trace_arguments,
                            solution_range=trace_solution_range,
                        )

                    post.export_report_to_jpg(
                        export_dir,
                        plot.plot_name,
                        width=image_size[0],
                        height=image_size[1],
                    )


    if __name__ == "__main__":
        export_circuit_reports(
            project_path=r"C:\\Projects\\demo\\example.aedt",
            design_name="Frequency",
            template_file=r"C:\\Users\\Public\\Documents\\Ansoft\\ReportTemplates\\NEXT.rpt",
            output_directory=r"C:\\Projects\\demo\\NEXT_Reports",
            trace_arguments=["16GHz"],
        )
